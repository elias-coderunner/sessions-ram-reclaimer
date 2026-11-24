# sessions-ram-reclaimer — technical details

> This file is needed to understand how the script works when editing it. It is not required for installing, operating, or running the solution.

Reaps idle Claude Code sessions on this host so their RAM goes back to the kernel while the transcript (when there is one) stays on disk.

## What it does

- Two reaping passes per run, in order:
  1. Resumable Claude Code sessions — any process with `--resume=<uuid>` in its cmdline.
  2. Bare `claude` processes hosted inside `tmux` (no `--resume` argument).
- Reads the optional `DO_NOT_KILL_FILE` (JSON array), merges its unexpired entries into the per-run skip lists, and prunes expired or malformed entries from the file atomically under a `.lock` sibling.
- For each candidate: sends `SIGTERM`, waits up to 5 seconds, then `SIGKILL` if still alive. For a tmux candidate, `tmux kill-session` goes first, with a `SIGTERM`/`SIGKILL` fallback aimed at the discovered `claude` pid; the stale `/tmp/tmux-<name>.pipe` is removed.
- Appends one summary line per run to `LOG`. `DRY_RUN=1` prints a preview to stdout and does not touch the log.

## How detection works

### Pass 1 — resumable sessions

Walks `ps -eo pid,args` for lines matching `--resume=`. For each:

1. Extracts the session UUID from `--resume=<uuid>` (36 hex-with-dashes).
2. Skips the pid if the UUID is in `EXCLUDED_SIDS` (from `SKIP_SESSIONS` + do-not-kill file).
3. Locates the transcript by filename: `find $CLAUDE_PROJECTS_DIR -maxdepth 3 -name "<uuid>.jsonl"`.
4. Reads the transcript's mtime via `stat -c %Y`. `idle = NOW - mtime`.
5. If `idle >= IDLE_THRESHOLD`, the pid is a candidate.

No transcript file → the pid is skipped (not reaped).

### Pass 2 — tmux-hosted bare `claude`

Only runs if `tmux` is on `PATH`. Walks `tmux list-sessions -F '#{session_name} #{session_activity}'`. For each session:

1. Skips if the session name is in `EXCLUDED_TMUX`.
2. `idle = NOW - session_activity`. If below `IDLE_THRESHOLD`, skip.
3. Gets the first pane's `pane_pid` for the session.
4. Expands that pid into the whole pane subtree via `pstree -p`; falls back to just the pane pid if `pstree` is missing.
5. Scans each pid's `/proc/<pid>/cmdline` for a cmdline whose argv0 matches `(^|/)claude( |$)` and has no `--resume=` anywhere. First match is the target.

No match → the tmux session is skipped.

### RSS accounting

For each reaped pid, `VmRSS` is read from `/proc/<pid>/status` before killing and summed into `RECLAIMED_KB`. If the pid was killed but the RSS reading was lost (`RECLAIMED_KB == 0` while `KILLED > 0`), the summary line reports `RAM reclaimed not captured` instead of `0 MB`.

## What is protected

| Category                                             | Mechanism                                                                                                                 |
|------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------|
| Server/bridge daemons under `~/.claude/remote/srv/`  | Implicit: no `--resume=<uuid>` argument, and not running inside `tmux`. Neither pass matches them.                        |
| `claude -p` / print-mode processes                   | Implicit: same as above — no `--resume=` and not under `tmux`. (They also exit on their own.)                             |
| Explicitly excluded sessions / tmux names            | `SKIP_SESSIONS`, `SKIP_TMUX`, and entries in `DO_NOT_KILL_FILE`. All are merged into `EXCLUDED_SIDS` / `EXCLUDED_TMUX` before the passes run. |

### `DO_NOT_KILL_FILE` format

JSON array. Each entry is a dict with:

- `type`: `"session"` or `"tmux"`.
- `id` (for `session`) or `name` (for `tmux`).
- `until`: ISO-8601 UTC timestamp, or the literal `"never"`. Naive datetimes (no `tzinfo`) are rejected to match the `/dontkillsession` helper's own write-side rejection.
- `reason`: optional free text.

Loading is wrapped in an `fcntl.LOCK_EX` on `<path>.lock` so prune and append cannot race. Expired or malformed entries are dropped on load and the file is rewritten atomically via `tempfile.mkstemp` + `os.replace`. If the top-level JSON cannot be parsed, one error line goes to `LOG` and the run continues with the env-var lists only.

## Config

Environment variables the script reads (defaults in the script itself):

| Variable              | Default                                                   | Role                                                                 |
|-----------------------|-----------------------------------------------------------|----------------------------------------------------------------------|
| `LOG`                 | `$SCRIPT_DIR/activity-log/sessions-ram-reclaimer.log`     | Where the per-run summary line is appended.                          |
| `IDLE_THRESHOLD`      | `21600` (6 h, in seconds)                                 | Age past which a session becomes a reap candidate.                   |
| `SKIP_SESSIONS`       | *empty*                                                   | Comma-separated session UUIDs the first pass must never kill.        |
| `SKIP_TMUX`           | *empty*                                                   | Comma-separated tmux session names the second pass must never kill.  |
| `DO_NOT_KILL_FILE`    | `$SCRIPT_DIR/do-not-kill-sessions.json`                   | JSON exclusion list; expired entries are auto-pruned.                |
| `DRY_RUN`             | `0`                                                       | `1` previews without killing and prints to stdout only.              |
| `CLAUDE_PROJECTS_DIR` | `$HOME/.claude/projects`                                  | Where the first pass searches for `<uuid>.jsonl` transcripts.        |

`SCRIPT_DIR` is resolved from `${BASH_SOURCE[0]}` at startup.

## Logging

One line per real run, appended to `LOG`:

```
2026-11-24 19:03:53  verification executed, 2 idle sessions detected, 2 killed, 531 MB of RAM reclaimed
```

Fields: timestamp, `verification executed`, `<N> idle sessions detected`, `<N> killed`, memory phrase (either `<X> MB of RAM reclaimed` / `<X.Y> GB of RAM reclaimed`, or `RAM reclaimed not captured` when RSS was lost). `, <N> failed` is appended only when `FAILED > 0`.

Additional lines, written only on runs that touch `DO_NOT_KILL_FILE`:

- `pruned <N> expired/malformed entry(ies) from <path>` — informational.
- `do-not-kill-sessions file <path> could not be parsed, ignoring` — one error line; the run continues with env-var lists only.

`DRY_RUN=1` prints a single preview line to stdout and writes nothing to the log — the log is an audit trail of actions taken, and a dry run takes none:

```
DRY-RUN PREVIEW: 2 idle sessions detected, would kill 2, would reclaim 531 MB of RAM
```

The script does not rotate its own log.
