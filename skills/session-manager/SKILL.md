---
name: session-manager
description: Use when session initialization context shows "SESSION NORMALIZER" with recent sessions listed, or when the user asks about sessions, merging, cleanup, or resume. On session start with prior sessions, this skill MUST push an AskUserQuestion resume prompt as the first user-facing action (after any mandatory skill bootstrap such as using-superpowers).
---

# Session Manager (v2.1)

This skill owns the mandatory resume/normalize UX for the session-normalizer plugin. When the SessionStart hook injects a `SESSION NORMALIZER` block listing ≥1 prior session for this project, this skill runs a blocking dialog before any normal task handling.

v2.1 adds: **project-grouped** top-level dialog (one option per project, not per session), a **second AskUserQuestion** for resume mechanics (Quit & relaunch vs Copy /resume vs Cancel), a **"real work" guard** on the current session, and **load-bearing ordering** of the Option-A hand-off.

## Ordering with mandatory-first-turn skills

If another mandatory-first-turn skill (e.g. `superpowers:using-superpowers`) must run on turn 1, run it FIRST. Then, **as the very next user-facing action**, execute the Turn-1 flow below. Never preempt `using-superpowers`. Never skip this flow in favor of answering the user's first prompt as a normal task — the user's prompt is addressed only after the resume dialog resolves.

## Turn 1 — project-grouped resume dialog

The SessionStart context groups sessions by `project_key` (the first non-empty `cwd` seen in the session's JSONL; see `[v2.1 Zeus #1]`). One representative session per project — the latest-mtime one — is surfaced as a single AskUserQuestion option. The secondary "matching vs other" split from `session-start` is preserved: groups in the current project come first, then other recent projects.

Call `AskUserQuestion` exactly once with **≤4 options**. Ordering:

1. The current project's latest session (if any matching session exists), marked *Recommended*.
2. Up to 3 other recent projects, newest first.

Anything beyond 4 is reachable later via `/normalize-sessions`.

### Option shape (per project group)

- **`label`** = `basename(project_key)`. Never the absolute path. `[v2.1 Zeus #3]` If two groups collide on basename, disambiguate by appending ` (<parent-dir>/<basename>)` — still never the full absolute path.
- **`description`** must include, in order:
  1. The full project path (`project_key`).
  2. The representative session id.
  3. The message count for that representative session.
  4. Group size: `N sessions in this project` (mandatory per `[v2.1 Zeus #2]`, even when `N == 1`).
  5. `⚠ unfinished` marker if the representative session's activity log flags it unfinished.
  6. The "Last actions:" preview from the SessionStart context if present.

Keep any preamble to one line. No wall-of-text. The dialog is the first user-facing interaction.

If the user picks a project, proceed to **Turn 2 — resume mechanics**. If zero prior sessions exist for any project, skip this flow entirely and proceed normally.

## Turn 2 — resume mechanics (AskUserQuestion #2)

After the user picks a project (which resolves to its representative session id = `<sid>`), fire a SECOND AskUserQuestion with **exactly** these three options:

1. **`Quit & relaunch`** *(Recommended)* — description: `Exit Claude Code, then paste claude --resume <sid> to reopen this session cleanly`.
2. **`Copy /resume`** — description: `Stay in this process; I will print /resume <sid> for you to paste into the current CLI`.
3. **`Cancel`** — description: `Do nothing`.

Do NOT invent an "Option B" self-relaunching wrapper. It is explicitly rejected by the plan as too invasive.

## Option A — Quit & relaunch (load-bearing ordering `[v2.1 Zeus #4]`)

Execute these steps in exactly this order. Do not reorder. Do not parallelize.

1. **Verify target.** Confirm `~/.claude/projects/<slug>/<sid>.jsonl` exists and is non-empty. `<slug>` is the target session's `project_key` with `/` replaced by `-`. If missing, unreadable, or zero-length, re-prompt Turn 1.
2. **"No real work" guard.** Inspect the CURRENT session's JSONL. If it contains ≥1 *real* user message — i.e. a user message that is NOT a local slash-command invocation and NOT this plugin's own skill chatter (the Turn-1/Turn-2 AskUserQuestion round-trip) — AUTOMATICALLY fall back to **Option C** and tell the user verbatim:
   > Your current session has real work in it; I won't delete it. Use /resume instead.
   Then run Option C's steps. Never silently destroy work-in-progress.
3. **Soft-delete the CURRENT empty session.** `mv` its `.jsonl` + sidecar directory + `.activity.jsonl` into:
   ```
   ~/.claude/projects/<current-slug>/.trash/<YYYYMMDD-HHMMSS>-<current-sid>/
   ```
   Use `mv` only — never `rm`, never hard-delete. **Wait for the move to complete (exit 0)** before proceeding. `.trash/` is purged on the next clean startup for entries older than 24h.
4. **ONLY THEN print the hand-off.** Print verbatim:
   ```
   Run this in your shell:

       claude --resume <sid>

   Then Ctrl+D to exit the current CLI, or just close this terminal.
   ```
   Never print the hand-off before step 3's `mv` has completed. If the user exits before the soft-delete lands, the empty session is left uncollected — that is exactly the failure mode `[v2.1 Zeus #4]` exists to prevent.
5. Proceed to **Merge-others sub-step** (below).

## Option C — Copy /resume

1. **Verify target** (same as Option A step 1).
2. **"No real work" guard** (same as Option A step 2, but on fall-back this branch IS Option C — do not recurse).
3. **Soft-delete the CURRENT session** the same way as Option A step 3 and wait for `mv` exit 0. This is still safe: the user is staying in this process, but the session JSONL they are in becomes effectively orphaned the moment they paste `/resume <other-sid>`, so collecting it now is correct.
4. **Print the paste line** verbatim:
   ```
   Paste this into the CLI:

       /resume <sid>
   ```
5. Proceed to **Merge-others sub-step** (below).

## Option Cancel

Do nothing. Do not soft-delete. Do not print a hand-off. Return to normal task handling.

## Merge-others sub-step (applies to both Option A and Option C)

After the user has confirmed resume intent and the current-session soft-delete has completed, fire a THIRD AskUserQuestion:

- **`Yes, merge`** — summarize older sessions of the SAME project into `docs/SESSION-HISTORY.md` and soft-delete them.
- **`No, keep them`** — leave them on disk untouched.

On `Yes, merge`: invoke the `/normalize-sessions` flow with BOTH parameters:

- `excluded_session_id = <sid>` (the resume target — never merged, never deleted).
- `project_key = <picked project_key>` (the project the user picked in Turn 1).

**Deletion-set formula `[v2.1 Zeus #5]`:**
```
{ s | s.project_key == <picked project_key>
      AND s.session_id != <sid>
      AND s.session_id != <current active session id> }
```
Sessions in OTHER projects are never touched by this flow, even when the legacy `cwd`-child/parent predicate would have matched them. The existing per-session `[Zeus #8]` guard still fires inside `/normalize-sessions`.

## On resume into a session marked ⚠ unfinished

Detected via the SessionStart context (the hook flags any session whose `.activity.jsonl` is non-empty, older than 120s, and whose last line is not `{k:"stop"}`).

**Do not proceed to normal task handling.** Immediately:

1. Read the last ~10 entries from `~/.claude/projects/<slug>/<sid>.activity.jsonl`. Tolerate a trailing partial/garbage line (catch JSON decode error on the last line only).
2. Present them under the header:
   > ## What was in flight when the previous run ended
   Format as a short table or bullet list: timestamp, kind (`tool`/`prompt`/`agent`), name, target/description.
3. For each entry, ask the user whether it appears to have completed. Prefer a single consolidated AskUserQuestion when practical.
4. Only after the user confirms recovery is complete may you resume normal task handling.

## On `/normalize-sessions`

Delegate to the command — follow the instructions in `commands/normalize-sessions.md`. Always pass `project_key` when the invocation originated from a Turn-1 project pick, and pass `excluded_session_id` whenever a resume target exists. A project with exactly one session still reports "already normalized".

## On an ad-hoc session-merge request

If the user asks to merge/consolidate sessions outside the startup flow:

1. Read the relevant `~/.claude/projects/<slug>/*.jsonl` files.
2. Extract key decisions, features, bug fixes, architecture choices.
3. Write a consolidated summary to `docs/SESSION-HISTORY.md`.
4. Soft-delete the merged session files into `.trash/` (never `rm`).
5. Report what was merged and where it landed.

## Rules (contract invariants)

v2.1 invariants (new — listed first because they are load-bearing):

- **Dialog labels always use `basename(project_key)`.** Never the absolute path. Collisions are disambiguated by `(<parent-dir>/<basename>)`, never by splashing the full path into a short label.
- **The current session is soft-deleted ONLY after all three conditions hold:** (1) the user has explicitly confirmed resume intent in Turn 2, (2) the "no real work" guard on the current session has passed, and (3) the `mv` into `.trash/` has completed with exit 0. Never before the hand-off line prints, never after — the `mv` must be strictly between confirmation and the hand-off.
- **Merge-others is filtered by `project_key`.** Sessions in other projects are inviolable and MUST NOT appear in the deletion set, even when the legacy `cwd`-child/parent predicate would have matched them.

Pre-existing invariants (still in force):

- **NEVER merge or summarize a session the user is resuming.** It is being loaded live; its content is working context, not archival.
- **NEVER hard-delete a session.** Always soft-delete into `~/.claude/projects/<slug>/.trash/<ts>-<sid>/`. `.trash/` is purged on next clean startup for entries >24h old.
- **`/normalize-sessions` MUST be invoked with an explicit `excluded_session_id` parameter whenever a resume target exists.** That ID MUST NOT appear in the deletion set; the command aborts loudly on guard violation.
- **NEVER preempt `using-superpowers` or any other mandatory-first-turn skill.** They run first; the resume prompt is the next user-facing action.
- **NEVER block on the resume dialog if zero prior sessions exist for this project** — proceed normally.
- **Keep output terse.** One line of preamble maximum before the AskUserQuestion. No wall-of-text.
- **Legacy sessions with no `.activity.jsonl` are never marked ⚠ unfinished** — they just show no "last actions" preview.
- **One-session projects**: `/normalize-sessions` reports "already normalized" and exits.
