---
name: remember-user-fact
description: Save a notable fact about the user to the configured personal-memory backend the moment you learn it — preferences, stable context, corrections to your behavior, recurring project details. Use immediately when the user says "remember that…", corrects you, or shares something that would help a future session. Routes to personal or work store by the deduction rule in CONTEXT.md. Default backend is the `memory` upstream on the MCP gateway (tools mcp__gateway__memory__*); a workspace .claude/memory-config.md can name another.
---

# Remember a user fact

When you learn something about the user that would be useful next session, save it **now** — don't defer to end-of-session. Small, atomic saves are easier to retrieve and update than one big dump later.

## When to invoke

- **Explicit request**: "remember that…", "save this for next time", "store in memory".
- **Correction**: the user tells you to change your behavior ("don't do X", "always do Y", "stop assuming Z"). These are high-value — save them so the next session doesn't repeat the mistake.
- **Confirmed preference**: the user states a stable preference with reasoning, not just an offhand comment. Reasoning is the signal that it's durable.
- **Durable project context**: you learn a client name, ongoing initiative, long-running constraint, or recurring stakeholder that will matter beyond this session.
- **Identity/role details**: role changes, new responsibilities, location changes, tooling switches.

Do **not** invoke for:

- Transient conversation state (current file, current task).
- Information already in code, git, or project docs — memory is for facts you cannot derive from the filesystem.
- One-off frustrations or snap opinions the user may not hold tomorrow.
- Secrets, credentials, tokens, or anything sensitive.

## How to run it

1. **Load the config** — read `.claude/memory-config.md` in the workspace. That file names the backend, the exact MCP tool to call for "add", and the scope parameters (namespace/project_id/etc.) for the chosen context. If the file is missing, use the default backend — the gateway `memory` server, Option A of the plugin's `templates/memory-config.example.md` (`mcp__gateway__memory__remember` with an explicit `namespace`). If that isn't available either, stop and ask the user to install a config.
2. **Pick the context** — apply the deduction rule from `CONTEXT.md`. If unsure, ask one short question ("save this to personal or work memory?").
3. **Call the configured add tool** with the context's scope parameters. Construct the record according to the schema in `memory-config.md`; at minimum:
   - A self-contained natural-language statement of the fact, written from the user's perspective as if they were telling a future assistant. Example: `"I prefer pnpm over npm for new Node projects — npm's lockfile churn has burned me twice."` Include the *why* if it was given; future-you uses the why to judge edge cases.
   - An absolute `YYYY-MM-DD` date.
   - Optional metadata the config recommends (type=preference/correction/project-context, topic tags, etc.).
4. **Before saving, check for duplicates** — run a quick semantic search for the same topic using the backend's search tool. If a near-identical memory already exists, prefer an update over creating a second record. Duplicates pollute retrieval.
5. **Confirm briefly** — one short line: "Saved to personal memory." or "Saved to work memory." Do not quote the full memory back.

## Quality bar

- **One fact per memory.** "Prefers pnpm; based in Jerusalem; works on X" is three memories, not one. Atomic memories are retrievable; compound ones are not.
- **Write for retrieval, not for logs.** The memory will be embedded and searched — use the keywords someone looking for this info would actually use.
- **Include the why when you have it.** "Prefers pnpm because npm's lockfile churn burned them twice" retrieves and reasons better than "prefers pnpm."
