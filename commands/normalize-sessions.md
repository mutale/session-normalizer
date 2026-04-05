---
allowed-tools: Bash, Read, Write, Glob, Grep, Agent, AskUserQuestion
description: Clean up and merge old sessions for the current project. Summarizes old sessions into docs, keeps the latest, deletes the rest.
---

# Normalize Sessions

You are running session normalization for the current project directory.

## Steps

1. **Scan sessions**: Find all session JSONL files in `~/.claude/projects/` that match the current working directory. Also check unindexed JSONL files. A session matches if its `projectPath` is the current directory, a child of it, or a specific parent (not the home directory).

2. **Show the user**: List all matching sessions with their summary, date, message count, and session ID. Format as a table.

3. **Ask which to keep**: Use AskUserQuestion to ask which session to keep as the primary. Default suggestion: the most recent session with the highest message count.

4. **Summarize old sessions**: For each session being merged:
   - Read the JSONL file
   - Extract user messages (type=user) and assistant text responses
   - Identify: what was the task, what was implemented/fixed, key decisions
   - Write a concise summary

5. **Write docs**: Create or append to `docs/SESSION-HISTORY.md` in the project directory with the consolidated summaries. If the file exists, append a new dated section.

6. **Delete old sessions**: Remove the JSONL files and associated directories for merged sessions. Do NOT delete the kept session or the current active session.

7. **Clean indexes**: Update any `sessions-index.json` files to remove stale entries.

8. **Report**: Tell the user how many sessions were merged, what docs were written, and how many remain.

## Important

- Never delete the current active session
- Always write the summary BEFORE deleting session files
- Ask for confirmation before deleting
- If a project has only 1 session, report "already normalized" and exit
