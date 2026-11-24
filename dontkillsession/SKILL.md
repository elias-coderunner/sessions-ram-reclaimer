---
name: dontkillsession
description: Protect Claude Code sessions from Sessions RAM Reclaimer for a duration the user specifies. Accepts free-form requests ("keep this one for a day", "don't kill anything I worked on today", "protect watcher-1 forever") and writes entries to the reclaimer's do-not-kill-sessions.json.
---

You are running inside a Claude Code session on a Linux host. The user has invoked `/dontkillsession` to protect one or more Claude sessions from being reaped by Sessions RAM Reclaimer. Your job is to figure out exactly what to protect and for how long, then call the helper script that writes the entries to disk.

## What the user may say

The input is free form. Examples you will see:

- `/dontkillsession` (no args) — they want to protect the current session; ask for how long.
- `/dontkillsession 24h` — protect the current session for 24 hours.
- `/dontkillsession until Friday` — protect the current session until next Friday.
- `/dontkillsession never` — protect the current session permanently.
- `/dontkillsession today` — bulk; protect every session with activity today. Ask for how long if they did not say.
- `/dontkillsession this week for 1 week` — bulk; protect every session with activity in the last week, for a week.
- `/dontkillsession the agent called watcher-1 forever` — tmux session named `watcher-1`, permanent.
- Anything else in natural language. Interpret it. If ambiguous, ask one short plain-language question. Do not guess when a sensible interpretation is unclear.

## What to do, in order

### 1. Read the install config

Both files are optional. If missing, use the fallbacks:

- `~/.config/sessions-ram-reclaimer/path` — one line, the install directory of the reclaimer (where `do-not-kill-sessions.json` lives). Fallback: ask the user where the reclaimer is installed, then offer to run `./setup` to save the answer.
- `~/.config/sessions-ram-reclaimer/timezone` — one line, an IANA timezone name (e.g. `Europe/Lisbon`). Fallback: `timedatectl show -p Timezone --value` (returns an IANA name on systemd hosts). If `timedatectl` is unavailable and the file is missing, fall back to `UTC` and mention it once in the reply so the user knows "today" is being computed in UTC. Never use `date +%Z` — it returns an abbreviation (e.g. `WET`, `EST`), which is not valid for `TZ=...` and is not an IANA name.

### 2. Identify what to protect

Resolve the user's intent into one or more entries. An entry is either a **session** (identified by a session UUID) or a **tmux** entry (identified by a tmux session name).

**Current session detection** (used for `/dontkillsession` with no target, "this one", "this session", etc.):

Walk up the process tree from your own shell, through `/proc/<pid>/status` → `PPid`, stopping when you find:

- An ancestor whose cmdline contains `--resume=<uuid>` → that UUID is the current session. Entry type `session`.
- Otherwise, if `$TMUX` is set, run `tmux display-message -p '#S'` to get the current tmux session name. Entry type `tmux`.
- If neither: this session has no stable identifier. Tell the user and stop.

Prefer the UUID path when both apply. A sample detection loop:

```bash
pid=$PPID
while [ "$pid" != "1" ] && [ -r "/proc/$pid/cmdline" ]; do
    cmd=$(tr '\0' ' ' < "/proc/$pid/cmdline")
    sid=$(printf '%s' "$cmd" | grep -oE -- '--resume=[0-9a-f-]{36}' | head -1 | cut -d= -f2)
    if [ -n "$sid" ]; then
        echo "$sid"; break
    fi
    pid=$(awk '/^PPid:/ {print $2}' "/proc/$pid/status" 2>/dev/null)
done
```

**Bulk time windows** (`today`, `this week`, `since Monday`, `last 3 days`, etc.):

- Boundaries are computed in the configured timezone (from step 1). "Today" means from midnight in that timezone to now.
- "Started" / "worked on" / "active" are all measured by the transcript file's modification time (last-message time). The reclaimer uses the same metric for idle; stay consistent.
- Scan `$CLAUDE_PROJECTS_DIR` (default `~/.claude/projects/`) for `*.jsonl` files; pick files whose mtime falls in the window; derive the session UUID from the filename. Use `find "$CLAUDE_PROJECTS_DIR" -maxdepth 3 -name '*.jsonl'` — same depth the reclaimer uses, so the two agree on which transcripts exist.

### 3. Resolve the duration

- `24h`, `2 days`, `30m`, `1 week`, `forever`, `until Friday`, `until 2026-12-10`, etc. Parse free-form. Compute `until` as an ISO-8601 timestamp with explicit `Z` suffix (`YYYY-MM-DDTHH:MM:SSZ`), or the literal string `never`.
- **Never emit a naive datetime** (no `Z`, no `+HH:MM` offset). The helper and the reclaimer both reject naive values because their interpretation is ambiguous.
- If the user said nothing about duration, ask one question in their language: "For how long?"

### 4. Collisions: ask

For every entry you are about to write, check whether that session UUID (or tmux name) is already in `do-not-kill-sessions.json` with an unexpired `until`. If so, show the user the existing entry (`until`, `reason`) and ask:

> "This one is already protected until {date}. Replace with the new time, extend (keep whichever is longer), or skip it?"

For a bulk request with many collisions, offer "apply the same choice to all remaining collisions" — do not interrogate the user N times in a row.

**One `--on-collision` per helper invocation.** The helper applies a single collision mode to every `--add` it receives. To honor mixed per-entry choices, call the helper multiple times — once per group of entries that share the same chosen mode. Example: user says "replace for session A, extend for sessions B and C" → one invocation with `--on-collision replace --add A`, a second with `--on-collision extend --add B --add C`.

### 5. Write

Call the helper:

```
<install-dir>/dontkillsession/dont-kill-this-session \
    --file <install-dir>/do-not-kill-sessions.json \
    --on-collision <replace|extend|keep> \
    --add '{"type":"session","id":"<uuid>","until":"<iso-or-never>","reason":"<short>"}' \
    --add '{"type":"tmux","name":"<tmux-session-name>","until":"<iso-or-never>","reason":"<short>"}'
```

Entry shapes:
- `session` entries use **`id`** (the Claude session UUID).
- `tmux` entries use **`name`** (the tmux session name).

One `--add` per entry; multiple `--add` flags are allowed in one call. The helper handles the atomic locked read-append-write and is the only thing (besides the reclaimer's own pruning) that touches the file.

Set `reason` to a short human-readable string reflecting what the user said (e.g. `"user said: keep for a day"`). Keep it under 120 characters.

### 6. Report

Tell the user in plain language what you did:

- Which sessions are now protected.
- Until when, in their local timezone.
- If anything was skipped (collision choice "keep"), say so.

Keep it short. One or two lines unless the bulk request produced a lot of entries, in which case show a summary count and offer to list them.

## Do not

- Do not invent default durations. Ask.
- Do not overwrite existing entries without asking.
- Do not touch the reclaimer script or the `do-not-kill-sessions.json` file directly — always go through the helper.
- Do not warn about a missing timezone config. Silent fallback is intentional.
- Do not create or modify files under `~/.config/sessions-ram-reclaimer/` yourself — that is `setup`'s job. If config is missing, point the user at `./setup` or ask them once and offer to run it.
