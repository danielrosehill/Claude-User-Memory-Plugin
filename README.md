# claude-user-memory

A Claude Code plugin that gives Claude a **persistent, cross-session memory layer about the user**. Two isolated contexts — personal and work — so Claude can remember your preferences, role, ongoing projects, and corrections across sessions without mixing the two.

**Default backend: the `memory` upstream on the MCP gateway** (tools `mcp__gateway__memory__*`). The plugin ships the *contract* (save / recall / commit-learnings skills + a personal/work routing rule); when those tools are present it uses them with no config. Mem0 or anything else still works — point the plugin at it via a workspace config file.

Claude Code's built-in per-project memory is great for code context, but it forgets everything about *you* the moment a new session starts, and it has no sense of which facts belong to your personal life vs. your professional work. This plugin fixes both — without locking you into a specific memory vendor.

## What it does

| Skill | When it fires | What it does |
| --- | --- | --- |
| `recall-user-memory` | Claude is about to ask you something a returning collaborator would already know, or needs user-specific context to answer | Semantic search against the configured backend, then silently integrates the result |
| `remember-user-fact` | You state a preference, correct Claude, or share a durable fact | Atomic save to the configured backend, with duplicate-check before write |
| `commit-learnings` | End of session, handover, or "save what you learned" | Reviews the session, proposes candidate facts, shows them for approval, then batch-saves |

All three skills share a **deduction rule** for picking the right context: default to personal; switch to work only on explicit override, work-repo cwd, or unambiguously business conversation. See `CONTEXT.md` for the full rule.

## How context isolation works

The plugin requires that personal and work memory be *separated* at the backend — separate namespaces on the gateway `memory` server, different Mem0 projects, different accounts, whatever the backend supports. The routing rule in `CONTEXT.md` is a second layer of safety, not the only layer. Two separated stores means a personal fact cannot architecturally end up in the work store, regardless of what Claude decides in a given turn.

## Installation

### 1. Install the plugin

```bash
claude plugins install danielrosehill/Claude-User-Memory-plugin
```

### 2. Make a memory backend available as an MCP

- **Default — the `memory` upstream on the MCP gateway.** If your MCP gateway serves a `memory` upstream (tools `mcp__gateway__memory__recall`, `remember`, `update_memory`, `forget`, `list_memories`, `list_namespaces`, `create_namespace`), there is nothing else to set up. Personal and work are kept apart by namespace (`work` vs `core` / `personal` / topic namespaces).
- **Mem0** — create two projects in [app.mem0.ai](https://app.mem0.ai) (one personal, one work). Configure two MCP HTTP servers pointed at `https://mcp.mem0.ai/mcp` and OAuth each separately.

Any other vector DB or memory service with an MCP interface is fair game.

### 3. (Optional) Create `.claude/memory-config.md` in your workspace

Not needed for the default backend. For any other backend, the plugin reads this file to resolve save/recall to concrete MCP calls. Copy the template from the plugin (`templates/memory-config.example.md`) into your workspace's `.claude/` directory and fill in:

- Your backend's MCP tool names for search / add / update
- Scope parameters to pass on every call (namespace for the gateway `memory` server; project_id/user_id for Mem0; etc.)
- How personal and work contexts are separated
- The record schema you want to commit to

Without this file, the skills use the gateway `memory` server if its tools resolve; if they don't, the skills stop and ask you to create the file — they will not guess any other backend.

### 4. Restart Claude Code

The skills become active once the memory-config file is in place.

## Usage

Mostly you don't invoke these skills directly — Claude will reach for them when the situation fits. Things that trigger them naturally:

- "Remember that I prefer X." → `remember-user-fact`
- "What's my preferred X?" → `recall-user-memory`
- "Save what you learned this session." → `commit-learnings`
- Any correction you give Claude ("don't do X, do Y") → `remember-user-fact`
- Any question where Claude would otherwise ask you something it should already know → `recall-user-memory` first

To force a context override, say "save this to **work** memory" or "check my **personal** memory".

## Design notes

- **Why backend-agnostic?** Memory stores evolve faster than Claude Code plugins should churn. The contract (two contexts, save/recall/commit, the routing rule) is stable; the backend isn't. Decoupling lets users migrate between backends (the gateway `memory` server, Mem0, anything else) without touching the plugin.
- **Why two contexts rather than one?** Mixing a user's personal preferences into their business client history corrupts both stores. Hard separation at the backend, soft separation at the routing rule, no third option.
- **Why three skills rather than one?** Each skill has a different trigger signature. `recall` fires before answering; `remember` fires in the middle of a message when a fact lands; `commit-learnings` fires at session boundaries. Collapsing them into one skill would make the trigger description incoherent.

## Upgrading from 1.x

v1.x shipped `.mcp.json` with two hardcoded Mem0 HTTP entries and referenced `mcp__mem0-personal__*` tools directly in every skill. v2.0.0 removes both.

To upgrade:

1. Install v2.0.0 of the plugin.
2. Write a `.claude/memory-config.md` in each workspace that uses memory. If you want to stay on Mem0, the template's Option B reproduces the v1.x behaviour. On the gateway `memory` server (Option A) no config file is needed.
3. Remove the old `mem0-personal` / `mem0-work` MCP server entries that the 1.x plugin injected at project level — v2.0.0 does not ship them.

## Changes in 2.2.0

The gateway `memory` server is the default backend, used without a config file when its tools resolve. The Pinecone example config (old Option A) was removed; a Pinecone setup still works as a custom backend under Option C.

## License

MIT — see `LICENSE`.
