# dont-kill-this-session — technical details

> This file is needed to understand how the helper works when editing it. It is not required for installing, operating, or running the solution.

Python helper that appends entries to Sessions RAM Reclaimer's `do-not-kill-sessions.json`. Called by the `/dontkillsession` skill once it has resolved the user's request into concrete entries; the helper itself does not interpret natural language.

## CLI surface

```
dont-kill-this-session --file <path> [--on-collision replace|extend|keep] --add <json> [--add <json> ...]
```

- `--file` (required) — path to `do-not-kill-sessions.json`. The file does not need to exist; a missing file is treated as an empty list.
- `--on-collision` — behavior when an incoming entry collides with an existing one on the same key. Allowed values: `replace` (default), `extend`, `keep`. One mode applies to every `--add` in a single invocation; mixed per-entry choices are honored by calling the helper more than once.
- `--add` (required, repeatable) — one JSON object per flag, each describing one entry to add.

## Entry schema

```
{"type": "session", "id":   "<uuid>", "until": "<iso-or-never>", "reason": "..."}
{"type": "tmux",    "name": "<name>", "until": "<iso-or-never>", "reason": "..."}
```

- `type` — `session` or `tmux`.
- `id` — required when `type` is `session` (the Claude session UUID).
- `name` — required when `type` is `tmux` (the tmux session name).
- `until` — required. Either the literal string `"never"`, or an ISO-8601 timestamp with an explicit `Z` suffix or `+HH:MM` offset. Naive datetimes (no `Z`, no offset) are rejected on purpose, so the helper and the reclaimer cannot disagree on what the string means.
- `reason` — optional, free-form short string.

The composite key used for collision detection is `(type, id)` for sessions and `(type, name)` for tmux entries. An entry missing the required identifier for its type is rejected.

## Collision semantics

Applied per `--add` against the existing file contents:

- `replace` — the incoming entry overwrites the existing one at the same key.
- `extend` — the longer of the two `until` values wins. Comparison goes through `is_longer`: `"never"` beats any timestamp, and between two timestamps the later one wins. If the existing entry is longer it stays; otherwise the incoming entry overwrites it.
- `keep` — the existing entry stays, the incoming one is dropped.

Entries whose key is not already present are always appended, regardless of mode.

## Concurrency

The file is shared with the reclaimer's own pruning pass and with other helper runs. Each invocation:

1. Opens `<file>.lock` and takes an exclusive `fcntl.flock` on it (blocks until acquired).
2. Reads and rewrites `--file` under that lock.
3. Releases the lock.

The rewrite is atomic: a temporary file is created in the same directory via `tempfile.mkstemp`, written and flushed, then moved into place with `os.replace`. A reader that opens the file at any moment sees either the complete old contents or the complete new contents, never a partial write.

## Validation

Every `--add` value is parsed and validated before the lock is taken or the file is touched:

- Must be valid JSON and a JSON object.
- Must carry a valid `(type, id)` or `(type, name)` pair.
- Must carry an `until` field, either `"never"` or an ISO-8601 string that parses to a timezone-aware datetime.

If any `--add` fails validation, the helper exits with code 2 and the file is not modified.

## Exit codes

- `0` — all entries applied; a one-line summary per entry is printed to stdout (`added`, `updated`, `skip`, or `kept`).
- `2` — bad input (invalid JSON, missing required fields, naive or malformed `until`, unparseable existing file) or a failure during the atomic write.
