---
allowed-tools: Bash, Read, Write, Glob, Grep, Agent, AskUserQuestion, Skill
description: Project-grouped, resume-first session normalization (v2.1). Groups sessions by project_key (first non-empty cwd), asks which PROJECT to resume, then which session in that project, then delegates the resume UX to the session-manager skill. All other sessions in the picked project get summarized into that project's docs/SESSION-HISTORY.md and soft-deleted into .trash/. Sessions in OTHER projects are never touched.
---

# Normalize Sessions (v2.1 — Project-Grouped Resume-First)

## Modes

This command has three modes:

1. **`/normalize-sessions --migrate`** — one-time historical backfill of the parallel store. Instructions:
   - Enumerate every `*.jsonl` under `~/.claude/projects/*/`.
   - For each, invoke `segment-normalize --jsonl <absolute-path>` (via the Bash tool, calling `${CLAUDE_PLUGIN_ROOT}/hooks/run-hook.cmd segment-normalize --jsonl <abs-path>`).
   - Count files processed, segments emitted (parse `segment-normalize` stderr under `SESSION_NORMALIZER_DEBUG=1`), and projects touched.
   - Print progress every 10 files.
   - Idempotent: files whose mtime already matches `meta.json.source_mtimes` are fast-path skipped by `segment-normalize` itself.
   - End with a summary: `Migrated N sessions into M projects, parallel store size: K`.

2. **`/normalize-sessions --rebuild <project-slug>`** — force re-scan of one project.
   - Locate `~/.claude/project-sessions/<project-slug>/meta.json`, read its `source_mtimes`.
   - For each JSONL listed there, clear the entry and invoke `segment-normalize --jsonl <abs-path>`.
   - Useful when you suspect a project's index is stale or corrupt.

3. **`/normalize-sessions`** (no flags) — interactive cleanup (existing v2.1 flow, unchanged). See below.

You are running session normalization. Semantics: group every scanned session by its `project_key`, ask the user which **project** to resume, then (if that project has >1 session) which **session** within it, then delegate the actual resume UX to the `session-manager` skill. Every *other* session in the picked project is summarized into that project's `docs/SESSION-HISTORY.md` and soft-deleted into `.trash/`. Sessions in OTHER projects are never touched by this command, even if the legacy cwd-child/parent predicate would have matched them. Nothing is ever hard-deleted.

## Definitions

- **`project_key(s)`** = the **first non-empty `cwd`** seen while reading session `s`'s JSONL (single pass, early-exit on first hit). Per `[v2.1 Zeus #1]`. Never "most frequent", never "most recent". Sub-directory collapsing is explicitly out of scope: `/a/b` and `/a/b/c` are two distinct projects.
- **`current_sid`** = the session ID of the session in which this `/normalize-sessions` command is currently executing. Resolve it by reading the `$CLAUDE_SESSION_ID` env var if present; otherwise by taking the most recently-modified JSONL under `~/.claude/projects/<slug-of-cwd>/` whose mtime is within the last 60 seconds (the session that is writing right now). If neither resolves, abort with a loud error — the `[Zeus #8]` guard cannot be honored without it.
- **`slug(cwd)`** = `cwd` with every `/` replaced by `-` (Claude Code's existing project-slug convention).
- **`excluded_session_id` (`X`)** = the session the user chose to resume. Empty in the "None — just clean up" branch.
- **deletion set** per `[v2.1 Zeus #5]`:
  ```
  deletion_set = {
      s for s in all_sessions
      if s.project_key == <picked_project_key>
      and s.session_id != X
      and s.session_id != current_sid
  }
  ```

## Steps

### 1. Scan phase

Enumerate every session JSONL under `~/.claude/projects/**/*.jsonl` (include unindexed files). For each session:

- Derive `project_key` per definition above (first non-empty `cwd` in the JSONL).
- Collect: `session_id`, JSONL path, mtime, latest summary, user-message count, and an "unfinished" flag under the `[Zeus #9]` heuristic (`<sid>.activity.jsonl` exists & non-empty, mtime > 120s old, last parseable line's `k` is not `"stop"`). Legacy sessions without an activity log are NEVER marked unfinished.

Group the scanned sessions by `project_key`. For each project group, compute:

- `basename(project_key)` — primary display label.
- Full absolute `project_key` — shown on a sub-row next to "Last actions:" preview, indented. Never used as an AskUserQuestion option label.
- Sessions of the group, sorted by mtime **descending** (newest first).
- `group_size` = number of sessions in the group.
- Representative session = the newest in the group. Its summary + msg count + ⚠ marker (if applicable) is what the project row shows.

### 2. Top-level AskUserQuestion — "Which project do you want to resume?"

Call `AskUserQuestion` with the prompt **"Which project do you want to resume?"**. Build the option list as:

- Up to **3** project groups, chosen most-recent-first by their representative session's mtime. Each option's `label` = `basename(project_key)` **only** per `[v2.1 Zeus #3]`. If two projects collide on basename, disambiguate with `(<parent-dir>/<basename>)` — **never** the full absolute path in a label. Each option's description carries: full path, latest summary (≤60 chars), latest msg count, `(N sessions)` group-size hint (mandatory per `[v2.1 Zeus #2]`), and a ⚠ prefix if the representative is unfinished.
- One option `"Other project"` (drill-down) — only include if there are more than 3 project groups. On selection, re-prompt with the remaining groups (same label rules).
- One terminal option `"None — just clean up without resuming"`.

### 3. Second AskUserQuestion — "Which session in this project?"

**Skip this step entirely if the picked project has exactly 1 session** (single-session projects go straight to step 4 with that one session as `X`).

Otherwise, call `AskUserQuestion` with the prompt **"Which session in this project?"**. Options = every session in the picked project group, sorted most-recent-first by mtime. Each option's `label` = `<summary-60> | <msgs> | <sid-prefix>` where:

- `<summary-60>` = latest summary truncated to 60 chars.
- `<msgs>` = integer user-message count.
- `<sid-prefix>` = first 8 chars of the session ID.

The chosen session's ID becomes `X`.

### 4. Delegate resume UX to the session-manager skill

Once `X` is known, invoke the `session-normalizer:session-manager` skill with `X` as the resume target. The skill handles:

- The A (quit + relaunch with `claude --resume <X>`) vs C (print `/resume <X>` for the current process) sub-choice.
- The `[v2.1 Zeus #4]` Option-A ordering (verify target JSONL non-empty → soft-delete current empty session → *then* print the hand-off line) and the automatic fall-back to Option C if the current session already has ≥1 user message.
- The `[Zeus #8]` guard on the current session's own soft-delete.

`/normalize-sessions` itself MUST NOT print the `claude --resume` or `/resume` hand-off line — delegation to the skill is mandatory so there is exactly one implementation of that UX.

### 5. Merge phase (the actual "normalize" work)

Compute the `deletion_set` per the `[v2.1 Zeus #5]` formula above. **Sessions in other projects are never in this set**, even if they would have matched the legacy cwd-child/parent predicate.

**If the picked project has exactly 1 session → skip the merge phase entirely** and proceed directly to step 6. There is nothing to summarize or trash.

Otherwise, for each session `S` in `deletion_set`, in order:

1. **GUARD `[Zeus #8]` (pre-mutation re-assertion)**. Before any filesystem mutation for `S`, re-assert:
   ```
   S.session_id != X  AND  S.session_id != current_sid
   ```
   If this assertion ever fails for any `S`, **abort the entire command immediately** with a loud error (`"ABORT: [Zeus #8] guard violation — refusing to delete X or current_sid"`). Do NOT proceed to any subsequent session in the deletion set. No partial cleanup.

2. Read `S.jsonl`. Extract: user prompts (`type=user`), assistant text, key decisions (tool uses with obvious outcomes, agent dispatches, commits, test results).

3. **Docs-before-trash**. Append a dated section to `docs/SESSION-HISTORY.md` **inside the picked project's directory** (i.e. at `<picked_project_key>/docs/SESSION-HISTORY.md`), NOT the current cwd. Create the file and parent directory if missing. The section header MUST include the date, the session ID, and the user-message count. The body is a terse bullet summary of the session's intent + outcomes.

4. **If and only if the docs write in (3) succeeded**, soft-delete `S`: `mkdir -p ~/.claude/projects/<slug(S.project_key)>/.trash/<timestamp>-<S.session_id>/`, then `mv` `S.jsonl`, its sidecar directory (if any), and `<S.session_id>.activity.jsonl` (if any) into that trash directory. The slug is derived from `S.project_key`, **not** from the current cwd. Use `mv`, never `rm`.

5. **If the docs write fails**: skip the trash move for `S`, log a warning to the user in the final report, and continue to the next session in the deletion set. Never trash content that was not successfully summarized.

### 6. "None — just clean up without resuming" branch

If the user chose the terminal `"None — just clean up without resuming"` option in step 2:

- Set `X = None` (no resumed session).
- Derive `picked_project_key` from the **current cwd** (`project_key_of_current = cwd`).
- Apply the merge phase (step 5) with that `picked_project_key`, using `deletion_set` computed as `{ s | s.project_key == cwd AND s.session_id != current_sid }` (the `X != ...` clause is vacuously satisfied).
- Do NOT delegate to the session-manager skill — there is nothing to resume.
- The current active session is never in the deletion set (guarded by `s.session_id != current_sid`).

This gives the user a way to archive everything old in the current project without resuming anything.

### 7. 24-hour `.trash/` purge (all projects)

At the very end of the command — after steps 5/6 have finished, regardless of which branch ran — walk every `~/.claude/projects/*/.trash/*` entry across **all** project slugs and delete any entry whose mtime is older than 24 hours. This is best-effort; failures here are reported but do not abort the command.

### 8. Report

Print a terse one-line summary of the form:

```
X sessions summarized into <basename(picked_project_key)>/docs/SESSION-HISTORY.md · Y moved to .trash/ · Z skipped
```

Where:
- **X** = count of sessions for which the docs write succeeded.
- **Y** = count of sessions successfully `mv`-ed into `.trash/` (subset of X).
- **Z** = count of sessions skipped because the docs write failed.

If a resume target was chosen, also note that the `session-manager` skill has taken over the resume hand-off — do NOT re-print `claude --resume` or `/resume <X>` here; the skill owns that UX.

If the 24-hour `.trash/` purge removed anything, append ` · purged N stale trash entries` to the report.

## Important

- **Never** touch a session outside the picked project. The deletion set is strictly scoped by `project_key == picked_project_key`.
- **Never** touch the session the user chose to resume (`X`) or the current active session (`current_sid`). The `[Zeus #8]` guard is re-asserted before every single filesystem mutation.
- **Never** use `rm` on a session file. Soft-delete into `.trash/` only.
- **Always** write the docs summary BEFORE the trash move. Docs-first, trash-second. If docs write fails, skip the trash move.
- **Never** edit under `cache/`. The source-of-truth is this marketplace directory.
- **Never** print the `claude --resume` or `/resume` hand-off line yourself. Delegate to the `session-normalizer:session-manager` skill.
- Junk-deletion respects the activity log: a session with an `.activity.jsonl` containing ≥1 `tool` or `prompt` entry is NOT junk, even if its `.jsonl` has 0 user messages.
