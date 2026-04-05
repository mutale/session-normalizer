# Session Normalizer

A Claude Code plugin that keeps your sessions clean and organized.

## What It Does

- **Auto-cleanup**: Deletes empty, exit-only, and auto-generated junk sessions on every startup
- **Session awareness**: Shows relevant sessions for your current project when starting a new session
- **Session merging**: Consolidates old sessions into project documentation, keeping only the latest

## Installation

```bash
claude plugin install --dir /path/to/session-normalizer
```

Or from a Git repository:

```bash
claude plugin install session-normalizer@your-marketplace
```

## How It Works

### On Every New Session

The `SessionStart` hook automatically:

1. Scans all session files and deletes junk (0-message, exit-only, 1-msg auto-generated)
2. Finds sessions matching your current project directory
3. Passes the session list to Claude as context

Claude then briefly presents your recent sessions and asks if you want to resume one or start fresh.

### Manual Cleanup

Use the `/normalize-sessions` command to:

1. See all sessions for the current project
2. Choose which to keep
3. Summarize old sessions into `docs/SESSION-HISTORY.md`
4. Delete the old session files

## What Gets Auto-Deleted

- Sessions with 0 messages
- Sessions with only exit/goodbye commands (<=3 messages)
- 1-message auto-generated sessions (QA runners, test generators, blog writers)

## Cross-Platform

Works on macOS, Linux, and Windows (via Git Bash). No external dependencies beyond Python 3 (ships with all major OS).

## Requirements

- Claude Code CLI
- Python 3 (for session file parsing)
