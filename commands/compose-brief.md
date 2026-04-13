---
allowed-tools: Bash, Read, Glob, Grep
argument-hint: <project-slug>
description: Emit a project memory brief from ~/.claude/project-sessions/<slug>/history.jsonl with continuation framing. No preamble.
---

# Compose Project Memory Brief (v3.1.1)

**CRITICAL — no preamble rule (`[v3.1.1 Zeus #3]`):** Output the brief verbatim as your next message. Do NOT preface with "I'll compose the brief now" or "Here is your memory". Do NOT summarize what you're about to do. Do NOT narrate your tool calls. **Your entire response IS the brief below.** One or two acknowledging sentences AFTER the brief are fine but optional. The continuation illusion breaks the moment you speak before the `=== PROJECT MEMORY LOAD ===` line.

You are composing a project-memory brief for the project slug passed in `$ARGUMENTS`. The brief is the assistant's own working memory for a project they have been continuously working on — it is not a document, not a reference, not something to quote. The user's next message is turn N+1 of the project, not the start of a new conversation.

## Step 1 — Validate input

The argument `$ARGUMENTS` is a project slug (e.g. `Users-yoda2-projects-foo`). Resolve the parallel-store directory:

```
<store> = ~/.claude/project-sessions/$ARGUMENTS/
```

Read `<store>/index.json` and `<store>/project.json`. If either file is missing or unreadable, emit exactly ONE line:

```
error: no parallel store found for project slug '<slug>'
```

and STOP. Do not emit any framing, do not proceed.

## Step 2 — Read history.jsonl

Read `<store>/history.jsonl`. Parse each line as a JSON object. Each record has shape `{t, src_sid, seg, role, text, tools?, cwd?}`. Count total turns.

- If the file is missing OR zero parseable turns → emit exactly ONE line:
  ```
  No prior memory found for this project.
  ```
  and STOP.
- Otherwise continue.

Also read `project.json` to get `project_basename` and `last_active` for the framing header.

## Step 3 — Apply heuristic decay (`[v3.1 Zeus #2]` shape)

Zero LLM calls. Purely mechanical. Walk the history chronologically and bucket every turn into one of three tiers based on how far from the end of history it is:

### Tier A — verbatim (most recent)

The last **~300 turns** go in as-is. For each turn emit a compact rendering:

```
<role>: <text>
  tools: <tool1>, <tool2>, ...        (only if `tools` is present and non-empty)
```

Preserve turn order. Keep `text` as stored (already capped at 2 KB user / 8 KB assistant by the drain path).

### Tier B — per-segment summaries (middle)

Turns from ~300 back to ~1500 back: group by `(src_sid, seg)` segment boundary. For each segment, emit exactly one line of the form:

```
[<YYYY-MM-DD> seg <src_sid[:8]>/<seg>] <first_user_prompt[:200]> | files: <top5_files> | last: <last_user_prompt[:200]> | <msg_count> msgs, <duration>
```

Where:
- `first_user_prompt` = the first `role=user` turn in that segment (truncated to 200 chars).
- `last_user_prompt` = the last `role=user` turn in that segment (truncated to 200 chars).
- `top5_files` = the 5 most frequent `Write:<path>` / `Edit:<path>` targets across all assistant turns' `tools` arrays in that segment, joined by `, `. If fewer than 5, emit what exists.
- `msg_count` = total turns in the segment.
- `duration` = `last.t - first.t` rendered as a human span (e.g. `42m`, `3h`, `2d`).
- `<YYYY-MM-DD>` = date component of the segment's first turn's `t`.

### Tier C — one-line pointers (oldest)

Turns older than ~1500 back: group by segment. For each segment emit exactly one line:

```
<YYYY-MM-DD>: <first_user_prompt[:60]>…
```

Nothing else. This is the "fossil layer" — it exists so the assistant knows the project has deeper history, not so it can reason from it.

## Step 4 — Budget check (`[v3.1.1 Zeus #2]`)

Total output cap: **45 KB** (we assume the v2.1 dialog text is also in context; the brief + dialog together must leave room for the real conversation).

**Non-Latin multiplier:** for any segment whose text contains characters in the Hebrew range `\u0590-\u05FF` (or other non-Latin scripts), apply a **0.6× multiplier** to that segment's contribution to the budget — i.e. Hebrew segments are packed tighter because they cost more context per displayed character.

If the composed brief exceeds 45 KB:

1. Drop the **oldest** Tier A turns first (converting them to Tier B segment summaries).
2. If still over, shrink Tier B per-segment summaries (truncate `first_user_prompt` and `last_user_prompt` more aggressively — 100 chars, then 60 chars).
3. If still over, drop the oldest Tier B segments entirely (they survive as Tier C pointers only).
4. Never drop Tier C — it's already minimal and preserves the provenance trail.

## Step 5 — Emit with continuation framing

Now emit your response. It is **exactly** the block below, with the placeholders filled in. No preface. No "here is the brief". No commentary.

```
=== PROJECT MEMORY LOAD ===
You are continuing the <project_basename> project (last active <last_active_date>).
Below is your complete working memory from prior sessions. This is your
own memory — not a document, not a reference, not something to quote.
When the user speaks, they are continuing the conversation. Their first
message is the next turn after this memory, not the start of a new one.

<Tier C pointers, oldest first>

<Tier B per-segment summaries, chronological>

<Tier A verbatim turns, chronological>

=== END PROJECT MEMORY ===

The user will now speak. Respond as someone who has been working on
this project continuously.
```

Chronological order across the whole body: Tier C (oldest) → Tier B (middle) → Tier A (most recent). Within each tier, also chronological.

## Reminders

- Source of truth is `~/.claude/plugins/marketplaces/session-normalizer-local/`. Never read or edit under `cache/`.
- `$ARGUMENTS` is a slug, not a path. Do not try to `cd` to it.
- No LLM calls. No network. Pure read + format.
- If anything fails mid-compose, emit a single-line error and STOP — do NOT emit a partial framed brief.
- The `=== PROJECT MEMORY LOAD ===` and `=== END PROJECT MEMORY ===` fences are load-bearing — the skill and hook both look for them.
