# memory-config.md (template)

Copy this file into your workspace as `.claude/memory-config.md` and fill in the specifics of **your** memory backend. The `user-memory` plugin reads this file to resolve every save/recall operation to a concrete MCP call.

Two options are shown below. Option A — the `memory` upstream on the MCP gateway — is the default: if `.claude/memory-config.md` is missing **and** its tools resolve, the skills use Option A as written here. Pick one, delete the others, and customise to your account.

---

## Option A — `memory` upstream on the MCP gateway (default)

### Backend

- **Name**: `memory` (self-hosted semantic memory served through the MCP gateway).
- **Embedding**: server-side. You send plain text; the server embeds it.
- **Tools load on demand.** They are deferred, so load them with `ToolSearch` before the first call: `select:mcp__gateway__memory__list_namespaces,mcp__gateway__memory__recall,mcp__gateway__memory__remember,mcp__gateway__memory__list_memories,mcp__gateway__memory__update_memory`.

### MCP tools to call

| Operation | Tool | Key arguments |
| --- | --- | --- |
| Search | `mcp__gateway__memory__recall` | `query`, `namespaces` (list), `limit`, `min_score` |
| Add | `mcp__gateway__memory__remember` | `text`, `namespace`, `source`, `tags` |
| Update | `mcp__gateway__memory__update_memory` | `memory_id`, and only the fields that change (`text`, `namespace`, `tags`, `source`) |
| Delete | `mcp__gateway__memory__forget` | `memory_id` |
| List | `mcp__gateway__memory__list_memories` | `namespace`, `limit`, `offset` |
| Namespaces | `mcp__gateway__memory__list_namespaces` / `create_namespace` | — |

### Context scope

The store is split into **namespaces** by area of life. Call `list_namespaces` once per session to see the real names and descriptions — do not guess them. If your gateway ships a namespace guide (for example a `gateway-usage` skill with a `memory-namespaces` reference), that guide is authoritative over this table.

| Context | Namespaces | Notes |
| --- | --- | --- |
| Personal | `core` (always-true basics: identity, family, home), `personal`, and the topic namespaces (`career`, health, family, …) | Load `list_memories(namespace="core")` at the start of a personal conversation. Pick the narrowest namespace that fits. |
| Work | `work` | Business, clients, professional preferences. |

Separation is enforced by always passing `namespaces` to `recall` (never `null`, which searches every namespace) and an explicit `namespace` to `remember`. Never write a personal fact to `work` or vice versa. Create a new namespace only when none fits (`create_namespace`, lowercase letters, digits and dashes, with a description).

### Record construction

- `text`: one self-contained statement that makes sense with no surrounding conversation — "The user prefers pnpm over npm for new Node projects", not "prefers pnpm". Include the absolute `YYYY-MM-DD` date in the text when the fact is time-bound.
- `source`: where it came from, e.g. `user said 2026-10-21`.
- `tags`: optional, e.g. `["preference"]`, `["correction"]`, `["project-context"]`.
- Ids are assigned by the server; keep the one `remember` returns if you'll update it in the same session.

### Duplicate policy

`remember` already refuses a near-duplicate in the same namespace and returns the existing memory instead. If the fact has changed, call `update_memory` on that id rather than passing `allow_duplicate=true`.

---

## Option B — Your own backend

Any backend works. Fill in:

- MCP tool names for search / add / update
- Scope parameters the agent must pass on every call
- The record schema (field names, required fields, id convention)
- The personal-vs-work separation mechanism (separate accounts? separate indexes? separate namespaces?)
- A duplicate-avoidance rule
