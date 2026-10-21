# Shared context for claude-user-memory skills

All three skills in this plugin (`recall-user-memory`, `remember-user-fact`, `commit-learnings`) share the rules below. Read this file before performing any memory operation.

## Default backend, overridable per workspace

This plugin does **not** ship MCP server definitions. It defines the contract for a personal-memory layer. The **default backend** is the `memory` upstream on the MCP gateway (tools `mcp__gateway__memory__*`), configured as Option A in `templates/memory-config.example.md`. Any other backend — Mem0, a raw vector DB, something else — can be named per workspace instead.

To override the default (or pin it explicitly), the user (or an agent on their behalf) places a file at `.claude/memory-config.md` in their workspace specifying:

- **Backend name** (gateway `memory` / Mem0 / …)
- **Tool names** for search, add, update, (optional) delete — as literal MCP tool identifiers the agent can call
- **Scope parameters** to pass on every call — for the gateway `memory` server this is `namespace` (on write) / `namespaces` (on search); for Mem0 it's `project_id` / `org_id` / `user_id`
- **Personal vs. work separation** — how the backend distinguishes the two contexts (separate namespaces, separate Mem0 projects, etc.)
- **Record schema** — field names, required metadata, id conventions

A working template lives in `templates/memory-config.example.md` in this plugin. Copy it into the workspace and fill it in.

If `.claude/memory-config.md` is missing:

1. Check whether the default backend is present: load `mcp__gateway__memory__list_namespaces` via `ToolSearch` and call it. (These tools are deferred — not seeing them in the initial tool list proves nothing; a schema that resolves does.)
2. If it answers, use the default: Option A of `templates/memory-config.example.md`, as written.
3. If it doesn't, **stop and ask the user to provide a config** rather than guessing a backend. A wrong backend write is much worse than a paused skill.

## Two contexts: personal and work

Every memory operation targets exactly one of two contexts:

- **Personal** — life, preferences, hobbies, family, health, non-business tooling opinions
- **Work** — professional work, clients, business projects, commercial deliverables

They are **deliberately separate**. A personal preference must never leak into the work store, and vice versa. On the default backend it is enforced by namespace (`work` for work; `core`, `personal` and the topic namespaces for personal) and by always scoping search and write to an explicit namespace. Other backends specify their mechanism in the workspace's `memory-config.md`.

## How to pick the right context

Default to **personal**. Switch to **work** only if at least one of these is true:

1. **Explicit override** — the user said "work memory", "save this to work", "check work context", or similar. Explicit wins over everything else.
2. **Working directory signal** — the current `cwd` is clearly a work/business repository (e.g. under a path containing the user's business name, a client project folder, or a repo whose README/CLAUDE.md identifies it as professional work).
3. **Conversation topic signal** — the active conversation is unambiguously about the user's business, a client, or professional deliverables (not just "work-adjacent" topics like learning a framework for fun).

If none of the above apply, use **personal**.

If the signal is mixed or ambiguous, **ask the user** rather than guessing. One short question is cheaper than polluting the wrong store.

## What belongs in memory

**Save** (high signal):

- Stable facts about the user (role, location, tooling preferences, family, health constraints)
- Preferences and opinions they've stated (e.g. "I prefer X over Y because…")
- Recurring project context (ongoing initiatives, clients, long-running goals)
- Corrections the user made to your behavior ("don't do X, do Y instead")
- Non-obvious domain knowledge that you'd otherwise have to re-derive next session

**Do not save** (low signal, noise):

- Transient state from the current conversation (what file you just edited, current task progress)
- Anything already derivable from the code, git history, or file contents
- Sensitive credentials, API keys, private tokens, or secrets of any kind
- Opinions the user expressed once offhand but didn't commit to
- Anything the user said in frustration rather than as a stable preference

## Dates

Convert relative dates ("next week", "last month") to absolute `YYYY-MM-DD` form before writing. Relative dates stored verbatim become wrong the moment the session ends.

## When a memory turns out to be wrong

If a recalled memory contradicts current reality, trust the current reality. Update or delete the stale memory rather than acting on it. Memories are time-stamped claims, not eternal truths.
