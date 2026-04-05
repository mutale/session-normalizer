---
name: session-manager
description: Use when session initialization context shows "SESSION NORMALIZER" with recent sessions listed, or when the user asks about sessions, merging sessions, or session cleanup
---

# Session Manager

## On Session Start

When you see "SESSION NORMALIZER" context with a session list in your first turn:

1. **Briefly present** the relevant sessions (2-3 lines max)
2. **Ask**: "Continuing one of these, or starting fresh?"
3. If continuing: tell the user to type `/resume` and pick the session
4. If starting fresh: proceed with their request, suggest naming with `/name`
5. **Do not block** — if the user's first message is clearly a new task, address it AND mention the sessions briefly

## On `/normalize-sessions`

When the user runs the normalize-sessions command, follow its instructions.

## On Session Merge Request

When the user asks to merge or consolidate sessions:

1. Read the session JSONL files (`~/.claude/projects/*/*.jsonl`)
2. Extract key decisions, features built, bugs fixed, architecture choices
3. Write a consolidated summary to `docs/SESSION-HISTORY.md` in the project directory
4. Delete the old session JSONL files (keep only the latest meaningful one)
5. Report what was merged and deleted

## Rules

- Never show more than 5 sessions in the startup list
- Keep startup output to 3-4 lines — don't wall-of-text the user
- Group sessions by project topic, not directory path
- If only 1 session exists for the project, just mention it briefly
- If cleanup happened, mention how many were cleaned (one line)
