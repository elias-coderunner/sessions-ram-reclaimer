# Agents

Orientation for AI agents working in this repository. The project itself is already documented in depth elsewhere; this file's only job is to route you to the right place based on what you are trying to do.

## What this repo is

A cron-driven reaper that kills idle Claude Code sessions on a Linux host to reclaim RAM. One bash script (`sessions-ram-reclaimer`) does the reaping in two passes — resumable sessions detected by `--resume=<uuid>`, and bare `claude` processes inside `tmux`. A companion Claude Code skill (`dontkillsession/`) lets users protect sessions by writing entries to a shared JSON exclusion list the reaper reads on each run.

## Where to read next

| If you need to…                                                               | Read                                                                                |
| ----------------------------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| Understand the problem this solves and the solution approach                  | `README`                                                                            |
| Find any file in the repo                                                     | `Index`                                                                             |
| Install the reclaimer on a machine                                            | `installation-instructions`                                                         |
| Install the companion skill                                                   | `dontkillsession/installation-instructions`                                         |
| Understand how the reclaimer script works internally                          | `script-internals/sessions-ram-reclaimer.md`                                        |
| Understand how the skill's helper works internally                            | `script-internals/dont-kill-this-session.md`                                        |
| Change a user-tunable knob (idle threshold, skip lists, log path, projects dir, dry run, exclusion file path) | the env-var table in `installation-instructions` Step 2                             |
| Change the skill's behavior (what it accepts, how it resolves intent)         | `dontkillsession/SKILL.md`                                                          |
| Change the skill's helper (the atomic writer for the exclusion list)          | `dontkillsession/dont-kill-this-session`                                            |
| See the running reclaimer's log                                               | `activity-log/sessions-ram-reclaimer.log` (runtime, gitignored)                     |

## House rules

Non-obvious invariants that are easy to break. Check these before editing anything.

- No hardcoded author-specific defaults in the reclaimer. Every tunable (`IDLE_THRESHOLD`, `SKIP_SESSIONS`, `SKIP_TMUX`, `LOG`, `CLAUDE_PROJECTS_DIR`, `DO_NOT_KILL_FILE`, `DRY_RUN`) defaults either to a `SCRIPT_DIR`-relative path, a `$HOME`-relative path, or an empty/documented value — never to anything specific to one operator's machine. Do not reintroduce a hardcoded user, hostname, or absolute path as a default; a fresh install on another host must run correctly on defaults alone.
- State-file name is `do-not-kill-sessions.json`. Both the reclaimer and the helper agree on this name via the `DO_NOT_KILL_FILE` env var with a default of `<script-dir>/do-not-kill-sessions.json` on the reclaimer side, and `--file <path>` on the helper side (the skill passes `<install-dir>/do-not-kill-sessions.json`). Any new reader or writer must use the same filename and must accept an override; if you rename it in one place without renaming it everywhere, the skill and the reaper stop sharing state.
- All readers and writers of the exclusion list serialize via `flock` on `<file>.lock`. The reclaimer's prune pass and the helper's append both take `fcntl.LOCK_EX` on the sidecar lock file. Any new process that touches `do-not-kill-sessions.json` must take the same lock, or an interleaved prune and append will clobber each other.
- Never emit a timezone-naive ISO-8601 string for `until`. The helper rejects naive datetimes on write (`parse_until` returns `None` when `dt.tzinfo is None`) and the reclaimer treats naive values as expired on read (`load_exclusions` drops them). The only valid forms are an ISO-8601 string with explicit `Z` or `+HH:MM` offset, or the literal `never`. Emit `YYYY-MM-DDTHH:MM:SSZ`.
- Two detection passes, each with a precise shape. Pass 1 reaps processes whose cmdline contains `--resume=<uuid>`, keyed off the transcript file's mtime under `$CLAUDE_PROJECTS_DIR`. Pass 2 reaps bare `claude` processes under `tmux`, keyed off `session_activity`, and is guarded by `command -v tmux` — if `tmux` is not installed the pass silently no-ops. Bridge/server daemons under `~/.claude/remote/srv/` match neither pass (no `--resume`, not inside tmux) and are implicitly protected by that fact — do not add a third pass that would sweep them in.
- The reclaimer's prune drops expired AND malformed entries. On every run, `load_exclusions` reads `do-not-kill-sessions.json`, keeps only entries that are well-formed (dict with a valid `type` and the matching `id`/`name`) and whose `until` is `"never"` or parses as a timezone-aware ISO-8601 in the future, and atomically rewrites the file if anything was dropped. A broken entry does not persist across runs.
- The helper is the only sanctioned writer of `do-not-kill-sessions.json`. Its `--on-collision` flag (`replace` | `extend` | `keep`, default `replace`) is the full write contract — one mode per invocation, applied to every `--add`. Any new writer that bypasses the helper must replicate the flock, the atomic write, and the collision semantics, or the file's invariants degrade.
- Runtime state lives in `activity-log/` and `do-not-kill-sessions.json`. The log file path is `activity-log/sessions-ram-reclaimer.log` by default. Both paths are gitignored at the repo root (`.gitignore`). Do not commit log content, and do not commit a populated exclusion list.

## Verifying a change works

Three commands, same as `installation-instructions` Step 4:

```
DRY_RUN=1 <install-dir>/sessions-ram-reclaimer                       # preview, no kills
<install-dir>/sessions-ram-reclaimer                                  # real run
tail -5 <install-dir>/activity-log/sessions-ram-reclaimer.log         # should show a fresh "verification executed" line, no parse-error lines
```
