# Session Normalizer

A Claude Code plugin that keeps your sessions clean, pushes a resume prompt when you start a new one, and — new in v2 — records a crash-safe activity log per session so you can recover state after an unexpected CLI termination.

## What It Does

- **Auto-cleanup** — Deletes empty, exit-only, and auto-generated junk sessions on every startup. Sessions that crashed before their first user prompt are now protected from this cleanup if they have activity-log content.
- **Push-on-start** — On a fresh session in a project that already has prior sessions, the `session-manager` skill immediately asks which one to resume (or start fresh). One keystroke and the dialog appears.
- **Resume-and-merge-others** — Pick a session to resume; the others get summarized into `docs/SESSION-HISTORY.md` and the current empty session is moved to `.trash/`. The resumed session is never touched.
- **Crash-safe activity log** — Every tool call, prompt, and turn-end is appended to a per-session JSONL log. After an unexpected shutdown (SIGKILL, power loss, OS update), the next startup marks the previous session ⚠ unfinished and surfaces the last ~10 actions so you can recover.
- **Soft-delete, not hard-delete** — Nothing is `rm`'d. Everything goes to `~/.claude/projects/<slug>/.trash/<timestamp>-<sid>/` and is purged on the next clean startup after 24 hours.

## Installation

```bash
claude plugin install --dir /path/to/session-normalizer
```

Or from a Git repository:

```bash
claude plugin install session-normalizer@your-marketplace
```

After install, reload Claude Code (quit + relaunch) so the plugin cache refreshes from the marketplace.

## How It Works

### On Every New Session

The `SessionStart` hook:

1. Deletes junk sessions (0-message, exit-only, 1-msg auto-generated) — unless the session has an activity log with real content, in which case it's preserved as a crash-recovery candidate.
2. Purges `.trash/` entries older than 24 hours.
3. Finds sessions matching your current project directory.
4. For each matching session, reads the last ~20 lines of its activity log and renders a `Last actions:` preview (e.g. `Write src/foo.ts · Bash pnpm · Task refactor-auth`).
5. Marks a session **⚠ unfinished** if its activity log exists, hasn't been touched in the last 120 seconds, and doesn't end with a clean `stop` event — these are crash candidates.
6. Injects a directive into the assistant's first-turn context telling it to call `AskUserQuestion` as its first user-facing action (after any mandatory skill bootstrapping such as `using-superpowers`).

Type anything (e.g. `hi`) to trigger the resume dialog.

### Activity Log

Per session, a JSONL file at `~/.claude/projects/<slug>/<session-id>.activity.jsonl` is maintained by three hooks:

- `UserPromptSubmit` (async) appends `{k:"prompt", p:<first 80 chars>}`
- `PostToolUse` (async, matcher `".*"`) appends `{k:"tool", n:<tool>, a:<target>, s:<status>}`
- `Stop` (**sync**) appends `{k:"stop"}`

Writes are single `write(2)` syscalls with `O_APPEND`, capped at 1024 bytes per event. This gives best-effort atomicity against concurrent writers but makes **no durability guarantee against power loss** — the read-back tolerates a trailing partial line. The log rotates in place when it exceeds 64 KB (keeps last 500 lines).

**Privacy note**: user prompts are persisted to disk unencrypted (first 80 chars). `Bash` command logging is limited to the first whitespace token only, to avoid capturing `export TOKEN=...` style secrets, but you should still avoid pasting sensitive data into prompts.

### Resume-and-Merge-Others Flow

When you pick "Resume session X" from the startup dialog (or via `/normalize-sessions`):

1. The target session's JSONL is verified readable.
2. Other sessions in the project are summarized into `docs/SESSION-HISTORY.md` (docs written **before** any file move).
3. Those other sessions + the current empty session are soft-deleted into `.trash/`.
4. A guard asserts that `excluded_session_id == X` is never in the deletion set; violation aborts the whole command.
5. You are instructed to run `/resume <X>`.

The resumed session is never summarized or touched. It is being loaded live, not archived.

### Crash Recovery

If SessionStart marks a session ⚠ unfinished and you choose to `/resume` it, the `session-manager` skill presents the last ~10 activity-log entries as "What was in flight when the previous run ended" and asks you to confirm which ones actually completed before proceeding with normal work.

### Manual Cleanup

Use `/normalize-sessions` to run the resume-and-merge-others flow on demand, or `/resume-smart` to re-trigger the resume push mid-session if you dismissed it initially.

## Hooks Registered

| Event | Matcher | Async | Script |
|---|---|---|---|
| `SessionStart` | `startup` | sync | `session-start` |
| `UserPromptSubmit` | — | async | `activity-log prompt` |
| `PostToolUse` | `".*"` | async | `activity-log tool` |
| `Stop` | — | **sync** | `activity-log stop` |

`Stop` is deliberately synchronous so the "clean exit" signal is durable — this is what crash-detection uses to distinguish a normal turn-end from a SIGKILL.

## Cross-Platform

Works on macOS, Linux, and Windows (via Git Bash). No external dependencies beyond Python 3.

## Requirements

- Claude Code CLI
- Python 3 (for session file parsing and activity-log event encoding)

## Changelog

### v3.1.0 — history drain, background migration, drain-on-start fallback

v3.1.0 is the **infrastructure half** of the v3.1 redesign. User-facing UX changes (project guess + offer card + memory-load framing) ship in v3.1.1 on top of this proven base.

- **New `history.jsonl` per project** in the parallel store (`~/.claude/project-sessions/<project-slug>/history.jsonl`). Append-only JSONL carrying the actual turn text from every drained session — user prompts capped at 2 KB, assistant turns at 8 KB (preserves inline text between tool calls; strips thinking blocks and base64 images). Written via the same single-`write(2)` + `O_APPEND` discipline as the v2 activity log.
- **Stop-hook now drains text**, not just metadata. `segment-normalize` walks the just-ended session's JSONL a second time after boundary detection, extracts per-turn `{role, text, tools}` records, and appends them to the target project's `history.jsonl`. On successful drain, the raw JSONL is soft-deleted into `~/.claude/projects/<slug>/.trash/<ts>-<sid>/` (30-day grace window). Skipped if any drain step errors, if the JSONL was touched within the last 60 s (active session), or if the soft-delete target already exists.
- **`project.json` per project** carrying `project_key`, `project_basename`, `last_session_name` (read from `~/.claude/sessions/*.json`), `last_active`, and `session_names_seen`. Populated on every drain. `color` field is reserved but always `null` because Claude Code does not persist session color.
- **`meta.json.history_drained` fast-path guard.** v3.0's mtime fast-path is now combined: a session is skipped only if `source_mtimes[src_rel] == src_mtime` AND `history_drained[src_rel] == true`. Existing v3.0 entries automatically re-walk on first v3.1.0 run to populate their history.
- **Background migration daemon.** On first SessionStart after v3.1.0 installs, the hook forks a fully-detached background process (double-fork + setsid + SIGHUP ignore + stdio redirected to `.migration.log`) that walks `~/.claude/projects/*/*.jsonl` and invokes `segment-normalize --jsonl` per file. Completes in the background without blocking the user. Writes `.migration-v3.1-done` sentinel on clean exit; stale `.migration-v3.1-running` sentinels older than 10 minutes trigger a re-launch on next SessionStart.
- **Drain-on-start fallback for unclean exits.** If `Stop` did not fire (system shutdown, power loss, SIGKILL), the next SessionStart runs the drain **synchronously** (up to a 10-second budget) for any raw JSONL lacking a recorded drain — not just warn. Closes the crash-gap that otherwise breaks the "ephemeral raw, canonical store" promise.
- **Per-project `.drain.lock`** at `~/.claude/project-sessions/<slug>/.drain.lock`. Prevents SessionStart's drain-on-start from racing a live Stop-hook drain. SessionStart polls up to 2000 ms with 50 ms intervals; stale PIDs are considered released.
- **UX unchanged.** SessionStart still shows the v2.1 project-grouped dialog. v3.1.1 will replace it with the project-guess + offer-card memory-load flow.

### v3.0.0 — infrastructure for project-first model

v3.0 is the **infrastructure batch** of the v3 redesign. No user-facing UX changes yet; those ship in v3.1.

- New **parallel store** at `~/.claude/project-sessions/<project-slug>/` (`index.json`, `meta.json`, future `briefs/`, future `resources/`). Plugin-owned, separate from Claude Code's `~/.claude/projects/` tree. Writes use per-file `os.replace` with `meta.json` as the sentinel; `fsync` before rename; reads tolerate missing files. `~/.claude/projects/` is read-only to the plugin — only the v2 activity-log sidecar lives under it.
- New **`hooks/segment-normalize`** helper: walks a session's JSONL linearly, computes `project_key` by searching upward up to 4 levels from each message's `cwd` for a project marker (`.git`, `package.json`, `Cargo.toml`, `pyproject.toml`, `go.mod`, `pom.xml`, `*.xcodeproj`), emits segment boundaries on `project_key` changes (not raw `cwd`), collapses 1–2-message blips between same-project neighbors, and upserts each touched project's entry. mtime fast-path short-circuit at entry makes reruns ~1 ms. Darwin-aware case normalization (`os.path.realpath` + `.lower()` for comparison, original case preserved for display).
- New **Stop-hook incremental normalization**: `hooks.json` now has a second `Stop` entry invoking `segment-normalize` alongside the existing `activity-log stop` entry. Every session close updates the parallel store in-process, fail-silent.
- New **`/normalize-sessions --migrate`** flag: one-time historical walk of `~/.claude/projects/*/*.jsonl` to backfill the parallel store. Idempotent — re-running is safe and fast because `segment-normalize`'s fast-path skips already-processed files.
- New **`/normalize-sessions --rebuild <project-slug>`** flag: force re-scan of a single project's segments.
- New **SessionStart delta-check warning**: if any JSONL under the current slug is newer than the parallel store's last-seen mtime for that session, the hook prepends `⚠ N session(s) pending normalization — run /normalize-sessions --migrate to import` to the output. Detects crash-gap situations where `Stop` didn't fire.
- Unchanged: v2.1 session-grouped resume dialog, activity log, `/resume-smart`, soft-delete to `.trash/`, Zeus governance, all the v1/v2 cleanup heuristics. The project-first dialog + brief composition ship in v3.1.

### v2.1.0

- Project-grouped dialog: the startup resume push now lists one row per **project** (not per session), showing `basename(project_key)`, the full path on a sub-row, the latest session's summary, message count, group size (`N sessions`), and the last-actions preview. Primary labels never leak the absolute path; basename collisions disambiguate with a parent-dir hint in square brackets.
- Every row carries context: project name + base path now appear alongside the session summary, so ambiguous first-prompts like `hi` become legible (you see which project the session belongs to and how many sessions are in it).
- Resume UX sub-choice: after picking a project, the skill now asks HOW to load it — **Quit & relaunch** (exit and paste `claude --resume <sid>`) or **Copy /resume** (paste into the current process). Option ordering for "Quit & relaunch" is load-bearing: verify target → guard against destroying a current session that already has real work → soft-delete the empty current session → THEN print the hand-off line.
- Merge-others is now filtered by `project_key`: `/normalize-sessions` only summarizes + soft-deletes sessions of the **picked project**, never sessions belonging to other projects, even if the legacy `cwd`-child/parent predicate would have matched them. The `excluded_session_id` guard still fires per-session.
- `project_key` derivation is deterministic: the **first non-empty `cwd`** seen in a session's JSONL. Stable across runs, cheap, matches how humans label sessions.
- `docs/SESSION-HISTORY.md` now lives in the **picked project's** directory, not the current cwd.
- The existing "matching vs. other projects" split is preserved as a secondary view on top of the project grouping.

### v2.0.0

- New: per-session activity log (`<sid>.activity.jsonl`) via `PostToolUse` / `UserPromptSubmit` / `Stop` hooks.
- New: crash-detection — sessions whose log doesn't end with `stop` and is >120 s old are marked ⚠ unfinished on next startup.
- New: `session-manager` skill pushes an `AskUserQuestion` resume dialog on turn 1.
- New: resume-and-merge-others flow with `excluded_session_id` guard — the resumed session is never touched.
- New: soft-delete to `.trash/` instead of hard delete; 24 h purge.
- New: `/resume-smart` command to re-trigger the resume push mid-session.
- New: `.zeus.json` for SDLCOlympus Zeus governance integration (production tier).
- Changed: junk-session cleanup now respects activity-log content (protects crashed-before-first-prompt sessions).
- Changed: Windows argv forwarding in `run-hook.cmd` is now quoted (`"%~2".."%~9"`).
- Docs: README rewritten.

### v1.0.0

- Initial release: junk-session auto-cleanup, project-relevant session listing, `/normalize-sessions` command.
