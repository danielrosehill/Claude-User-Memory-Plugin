---
name: commit-learnings
description: End-of-session sweep — review what you learned about the user during the conversation and commit durable facts to the configured personal-memory backend. Use when wrapping up a session, when the user says "save what you learned" or "commit this to memory", or proactively before a handover. Complements remember-user-fact (which saves in-the-moment); this skill catches facts that slipped through. Default backend is the `memory` upstream on the MCP gateway (tools mcp__gateway__memory__*); a workspace .claude/memory-config.md can name another.
---

# Commit end-of-session learnings

Over a long session, facts accumulate that weren't important enough to trigger `remember-user-fact` in the moment but, in aggregate, amount to a meaningful update to what memory knows about the user. This skill is the sweep — a structured review before the session ends.

## When to invoke

- User says "save what you learned", "commit this to memory", "update memory", or similar.
- User is wrapping up a session (handover, `/clear`, end-of-day).
- Proactively, when a long session has clearly surfaced durable facts you haven't saved yet.
- Before a handover document is written — the next session should inherit the learnings.

Do **not** invoke:

- In short sessions where nothing new was learned about the user.
- When the only "learnings" are transient task state (what got done this session belongs in a handover doc, not memory).

## How to run it

### 1. Load the config

Read `.claude/memory-config.md` in the workspace. You will need the backend's search tool (for the duplicate check) and add/update tool (for the writes), plus scope parameters for both personal and work contexts. If the file is missing, use the default backend (the gateway `memory` server — see `CONTEXT.md`); if that isn't available either, stop and ask the user to install a config.

### 2. Scan the session

Walk back through the conversation and list every candidate fact. For each, ask:

- Is it about **the user** (preferences, role, context, corrections) rather than the code or task?
- Is it **durable** — still true next week, next month?
- Is it **not already in memory**? Quickly search to check.
- Is it **not sensitive** (no credentials, tokens, private third-party info)?

A fact that passes all four is a commit candidate.

### 3. Group by context

Separate candidates into personal and work piles using the deduction rule in `CONTEXT.md`. A single session can produce commits to both stores — that's fine, just keep them separate.

### 4. Show the user before saving

This skill is different from `remember-user-fact` because it's batch and retrospective — the user may not remember telling you half of this. Before saving, show the proposed commits:

```
I'd like to save the following to memory:

PERSONAL (2):
  • <fact 1>
  • <fact 2>

WORK (1):
  • <fact 3>

Proceed? (you can edit, skip items, or reclassify)
```

Respect edits and skips. Never save a fact the user rejected.

### 5. Save

For each approved fact, call the configured add tool with the right scope parameters (see `remember-user-fact` SKILL.md for the per-record construction). Before each save, do a quick duplicate check via the configured search tool; if a close match exists, prefer an update over creating a duplicate.

### 6. Report

One short summary line: "Saved 2 to personal, 1 to work." Do not re-list the facts.

## Quality bar

- **Atomic commits.** One fact per memory. Do not concatenate the whole session's learnings into one blob.
- **Be stingy.** A session that saves 10 memories is usually a session with 8 false positives. Three well-chosen commits beat ten noisy ones.
- **If in doubt, skip.** You can always save a fact next session when it comes up again. A missing memory is cheaper than a wrong one.
