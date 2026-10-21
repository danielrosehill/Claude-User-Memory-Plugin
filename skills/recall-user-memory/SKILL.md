---
name: recall-user-memory
description: Retrieve facts about the user from the configured personal-memory backend when you suspect memory holds context you need — preferences, past decisions, ongoing projects, personal details. Invoke proactively any time you would otherwise ask the user something that feels like it should already be known, or when starting a task where prior context would help. Routes to personal or work store by the deduction rule in CONTEXT.md. Default backend is the `memory` upstream on the MCP gateway (tools mcp__gateway__memory__*); a workspace .claude/memory-config.md can name another.
---

# Recall user memory

Before answering the user or making a recommendation, ask yourself: **is this something memory might already know?** If yes, check it first.

## When to invoke this skill

- The user asks a question whose best answer depends on their preferences, role, tooling, or prior decisions.
- You are about to ask the user something that a returning collaborator would already know ("what's your preferred X?", "which framework do you use?", "what time zone are you in?") — check memory first; only ask if memory comes up empty.
- A new task starts and you suspect ongoing-project context exists (recurring clients, long-running initiatives, persistent constraints).
- The user references something with "as I mentioned before" or "you know how I…" — they are telling you memory should have it.

Do **not** invoke for purely technical questions with no user-specific answer (e.g. "what does this Python syntax do?").

## How to run it

1. **Load the config** — read `.claude/memory-config.md` in the workspace. That file names the backend, the exact MCP tool to call for "search", and the scope parameters (namespace/project_id/etc.) for the chosen context. If the file is missing, use the default backend — the gateway `memory` server, Option A of the plugin's `templates/memory-config.example.md` (`mcp__gateway__memory__recall` with an explicit `namespaces` list). If that isn't available either, stop and ask the user to install a config.
2. **Pick the context** — apply the deduction rule from `CONTEXT.md` (default personal; switch to work on explicit override, work cwd, or clearly-business conversation).
3. **Call the configured search tool** with:
   - The context's scope parameters from `memory-config.md`.
   - A short, specific natural-language query. "Preferred Python package manager" beats "tools".
4. **Integrate silently** — use the result to shape your answer. Do not announce "I recalled from memory that…"; just work with the corrected context. If a recalled fact conflicts with what you're observing now, trust observation and flag the stale memory for update (invoke `remember-user-fact` to overwrite).
5. **If memory is empty** — proceed as usual, then consider whether the answer you land on is worth saving (`remember-user-fact`) or queueing for end-of-session (`commit-learnings`).

## Quality bar

- One search per distinct question, not a scattershot of queries. If the first search returns nothing useful, a second phrasing is fine; a third is usually wasted.
- Only retrieve from one context at a time. Crossing the personal/work boundary defeats the point of having two stores.
