---
allowed-tools: Bash, Read, Write, Glob, Grep, Agent, AskUserQuestion
description: Re-trigger the session-manager resume flow mid-session.
---

# Resume Smart

Invoke the `session-manager` skill's resume flow now. Present the user with the same `AskUserQuestion` resume/normalize options that `SessionStart` would have shown on turn 1 — including any `⚠ unfinished` markers and activity-log "last actions" previews. Defer all actual state mutation (soft-deletes, summaries, `.trash/` housekeeping, `excluded_session_id` guard enforcement) to that skill; this command is purely a trigger.
