# memory-config.md (template)

Copy this file into your workspace as `.claude/memory-config.md` and fill in the specifics of **your** memory backend. The `user-memory` plugin reads this file to resolve every save/recall operation to a concrete MCP call.

Two sample configurations are shown below (Pinecone and Mem0). Pick one, delete the other, and customise to your account.

---

## Option A — Pinecone (vector DB, bring-your-own-index)

### Backend

- **Name**: Pinecone
- **Embedding model**: whatever your index was created with (e.g. `llama-text-embed-v2`, `text-embedding-3-small`). The index's integrated embedder handles vectorisation from the `text` field; you do not embed client-side.

### MCP tools to call

Set these to the actual tool identifiers your Pinecone MCP exposes. Examples below assume the Pinecone MCP is registered as `pinecone` via MCP Jungle under the `personal` aggregator, so tools look like `mcp__jungle-personal__pinecone__*`. Adjust to match your install.

| Operation | Tool |
| --- | --- |
| Search | `mcp__jungle-personal__pinecone__search-records` |
| Add / update | `mcp__jungle-personal__pinecone__upsert-records` |
| Inspect | `mcp__jungle-personal__pinecone__describe-index-stats` |

### Context scope

| Context | Index | Primary namespace | Also write to |
| --- | --- | --- | --- |
| Personal | `my-personal-index` | `foundational` (slow-changing truths) | `general-context` for state snapshots, `logs` for dated entries |
| Work | `my-business-index` | `foundational` | as needed |

The two contexts **must** be in different indexes (or at minimum, different namespaces) so a personal fact cannot accidentally land in the work store.

### Record schema

Every record:

```
_id: <stable-id-YYYY-MM-DD>        # e.g. housing-status-2026-04-23
text: <natural-language fact>      # the thing future-you will read back
category: personal | work | shared
date: YYYY-MM-DD                   # absolute, never relative
topics: [tag1, tag2, ...]          # optional, for filter queries
source: <workspace or repo name>   # optional, for provenance
```

### Duplicate policy

Before writing a new record, run a search on the same namespace with a query built from the fact. If a close match exists, overwrite it (same `_id`) rather than creating a second record.

### Namespace promotion

Daily log entries in the `logs` namespace are low-commitment. If a durable fact surfaces inside a log, also upsert a canonical record into `foundational` or `general-context` so it retrieves cleanly.

---

## Option B — Mem0 (managed memory SaaS)

### Backend

- **Name**: Mem0
- **MCP transport**: two HTTP MCPs, one per context (OAuth per project).

### MCP tools to call

| Operation | Personal tool | Work tool |
| --- | --- | --- |
| Search | `mcp__mem0-personal__search_memories` | `mcp__mem0-work__search_memories` |
| Add | `mcp__mem0-personal__add_memory` | `mcp__mem0-work__add_memory` |
| List | `mcp__mem0-personal__get_memories` | `mcp__mem0-work__get_memories` |
| Update | `mcp__mem0-personal__update_memory` | `mcp__mem0-work__update_memory` |

### Context scope

Pass on every call:

| Context | `project_id` | `org_id` (if paid tier) | `user_id` |
| --- | --- | --- | --- |
| Personal | `proj_xxx` | `org_xxx` (omit if unset) | `"default"` |
| Work | `proj_yyy` | `org_yyy` (omit if unset) | `"default"` |

### Record construction

Pass `messages` as a short, self-contained natural-language statement written from the user's perspective. Optional `metadata`: `{"type": "preference" | "correction" | "project-context"}`.

### Duplicate policy

Before adding, run `search_memories` on the same topic. If a near-identical memory exists, call `update_memory` instead of `add_memory`.

---

## Option C — Your own backend

Any backend works. Fill in:

- MCP tool names for search / add / update
- Scope parameters the agent must pass on every call
- The record schema (field names, required fields, id convention)
- The personal-vs-work separation mechanism (separate accounts? separate indexes? separate namespaces?)
- A duplicate-avoidance rule
