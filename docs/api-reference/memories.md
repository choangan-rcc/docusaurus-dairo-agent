---
sidebar_position: 14
title: Memories
---

# Memories

**Long-term memory** persists durable facts about a user across conversations.
Without it an agent only has working memory — the per-conversation message window
— so a preference stated on Monday is unknown on Tuesday.

Two independent paths, each with its own kill switch:

- **Read path** — synchronous, inside the turn. The user's message is embedded,
  the subject's memories are vector-searched, and a bounded `<memories>` block is
  injected into the system prompt. It **fails open**: any error or timeout logs
  and injects nothing rather than breaking the turn.
- **Write path** — asynchronous, in the background worker. After a turn, an
  extraction job is enqueued; it pulls candidate facts out of the new messages,
  embeds each one, searches existing memories, and lets the model decide
  ADD / UPDATE / NOOP. It never blocks or fails a turn.

This API is the **lifecycle surface**: list, export, delete, and switch learning
and reading on and off. Memories are never created through it — only extraction
writes them.

:::note Self-service only
Every route operates on the **authenticated caller's own** memories; the subject
is forced to the caller's user id and there is no way to name another subject.
Roles are stored but not yet enforced, so a cross-subject admin view would be
decorative — it is deliberately absent. The legacy Basic-auth principal is a
shared synthetic identity with no memory subject: these routes return `404` for
it.
:::

## Concepts

**Subject** — the person a memory is about, identified by the authenticated
user's id, always paired with a workspace. A memory can never be retrieved
outside its `(workspace, subject)` scope.

**Scope** — which agents may see a memory. Assigned **by code**, never by the
model:

| `scope` | Categories | Visibility |
| --- | --- | --- |
| `workspace` | `language`, `addressing`, `timezone`, `response_style` | Identity/profile facts — every agent in the workspace. |
| `agent` | everything else (`preference`, `decision`, `goal`, `entity`, `feedback`, `other`) | Only the agent that learned it. |

A misclassified or unknown category lands in the agent silo at worst, never in
the shared tier. An agent configured with `memory_isolated: true` writes and
reads only its own silo, whatever the category.

**Type** — `semantic` for a durable fact (a preference, constraint, role, or bit
of domain context) or `episodic` for a time-bound event or decision. Episodic
memories decay in relevance over time (halving roughly every two months), so
recent events outrank stale ones.

Extraction is explicitly forbidden from storing credentials, tokens, payment
data, government ids, health data, or precise addresses, and conversation text is
treated as untrusted data — facts are stored as plain text and never followed as
instructions.

## Memory object (`MemoryOut`)

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | Memory id (prefix `mem_`). |
| `scope` | string | `workspace` or `agent`. |
| `category` | string | The extraction category (see above). |
| `type` | string | `semantic` or `episodic`. |
| `content` | string | The remembered fact, one or two sentences. |
| `agent_id` | string \| null | The agent that learned it (`null` for workspace-tier facts). |
| `source_conversation_id` | string \| null | The conversation it came from — every memory is auditable back to its source. |
| `created_at` / `updated_at` | string | ISO-8601 timestamps. |

## `GET /v1/memories/me`

Everything the platform remembers about the caller in the active workspace.
Paginated — see [Pagination](/docs/api-reference/overview#pagination).

**Query parameters**

| Name | Type | Default | Description |
| --- | --- | --- | --- |
| `agent_id` | string | — | Only memories learned by this agent. |
| `type` | string | — | `semantic` or `episodic`. |
| `scope` | string | — | `workspace` or `agent`. |
| `q` | string | — | Substring match on `content`. |
| `sort` | string | `created_desc` | `created_desc` or `created_asc`. |
| `ids` | string | — | Comma-separated ids. The response preserves the **requested order** (used to resolve the ids in a trace); ids that no longer exist are simply absent. |

```bash
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8000/v1/memories/me?scope=workspace&limit=20"
```

```json
{
  "items": [
    {
      "id": "mem_4b8c1e70",
      "scope": "workspace",
      "category": "language",
      "type": "semantic",
      "content": "Prefers replies in Vietnamese.",
      "agent_id": null,
      "source_conversation_id": "conv_5d6e7f80",
      "created_at": "2026-07-28T09:12:00Z",
      "updated_at": "2026-07-28T09:12:00Z"
    }
  ],
  "total": 1,
  "limit": 20,
  "offset": 0
}
```

## `GET /v1/memories/me/state`

The caller's true learning and reading state — the platform flags, workspace
allowlist, worker configuration, and the caller's own switches resolved into
four booleans, so a UI never has to guess.

Also registered as `GET /v1/memories/state` (same handler).

| Field | Type | Description |
| --- | --- | --- |
| `memory_optout` | boolean | The caller used the right-to-forget flow and has not opted back in. |
| `opted_out_at` | string \| null | When that happened. |
| `subject_reading_enabled` | boolean | The caller's own "use what is already remembered" switch. |
| `learning_enabled` | boolean | Whether new facts **can** currently be learned: platform flag **and** workspace allowlist **and** an `arq` write path **and** not opted out. |
| `reading_enabled` | boolean | Effective read state: platform flags **and** the caller's switch. Per-conversation read toggles live on the conversation. |

```json
{
  "memory_optout": false,
  "opted_out_at": null,
  "subject_reading_enabled": true,
  "learning_enabled": true,
  "reading_enabled": true
}
```

## `GET /v1/memories/me/export`

Data portability: every active memory in one response, no pagination (bounded by
`MEMORY_MAX_PER_SUBJECT`, default 2000). Returns a **plain array** of memory
objects.

## `PUT /v1/memories/me/reading`

The subject-level **read** switch. Off means none of the caller's conversations
inject memories anywhere in this workspace — nothing is deleted and learning is
untouched. Also registered as `PUT /v1/memories/reading`.

**Request body:** `{ "enabled": false }`

**Response** `200` — the resulting state object (same shape as
`GET /me/state`).

## `POST /v1/memories/me/pause`

Stop **learning** without deleting anything. Existing memories stay listed and
keep serving turns. Nothing said while paused is ever learned — resuming
fast-forwards the extraction watermarks rather than back-filling the gap.

Also registered as `POST /v1/memories/pause`. **Response** `204`.

## `POST /v1/memories/me/resume`

Resume learning after a pause or a forget. Watermarks fast-forward to now, so the
paused period is never extracted. Also registered as
`POST /v1/memories/resume`. **Response** `204`.

## `POST /v1/memories/optin`

Re-enable memory after a right-to-forget. Same effect as `resume`: the opt-out
flag clears and watermarks fast-forward. **Response** `204`.

## `DELETE /v1/memories/{memory_id}`

Hard-delete one of the caller's own memories — the row and its embedding are
gone. **Response** `204`; `404` if the id isn't the caller's (ownership and
existence are indistinguishable by design).

## `DELETE /v1/memories`

Two behaviours on one route, selected by whether a body is present.

**No body — right to forget.** In one transaction: hard-delete **all** the
caller's memories in this workspace, fast-forward their extraction watermarks,
and set the opt-out flag. Learning stays off until `resume`/`optin`.

```bash
curl -H "Authorization: Bearer $TOKEN" -X DELETE \
  http://localhost:8000/v1/memories
```

```json
{ "deleted": 37 }
```

**Body `{"ids": [...]}` — targeted bulk delete.** Deletes exactly those
memories. Learning is not paused and nothing else is touched. An explicit empty
list deletes nothing — it is never treated as forget-all.

```json
{ "deleted": ["mem_4b8c1e70"], "missing": ["mem_deadbeef"] }
```

Ids that don't exist — or belong to someone else, the same outcome under the
mandatory subject filter — are reported as `missing`, never as an error.

There is no filterless wipe and no way to name another subject.

## `POST /v1/memories/me/bulk-delete`

Explicit-verb alias of the bulk-delete form above, for clients that strip request
bodies from `DELETE`. Same request (`{"ids": [...]}`) and same
`{deleted, missing}` response.

## Per-conversation and per-agent controls

Memory has switches at four levels — all four must allow it for a fact to be
read or learned:

| Level | Control |
| --- | --- |
| Platform | `MEMORY_ENABLED`, `MEMORY_READ_ENABLED`, `MEMORY_WORKSPACE_ALLOWLIST` |
| Subject (this API) | `pause`/`resume`, `PUT /reading`, opt-out |
| Agent | `memory_config`: `{ enable_memory, memory_isolated }` on the [agent object](/docs/api-reference/agents) |
| Conversation | `PUT /v1/agents/{id}/conversations/{cid}/memory/read` and `.../memory/write` — see [Chat](/docs/api-reference/chat#memory-toggles) |

## Configuration

| Env var | Default | Purpose |
| --- | --- | --- |
| `MEMORY_ENABLED` | `false` | Master switch (dark launch). Off = both paths are no-ops. |
| `MEMORY_READ_ENABLED` | `true` | Kill switch for prompt injection only; extraction keeps running. |
| `INGEST_QUEUE` | `inprocess` | Must be `arq` for the write path — under `inprocess` extraction is a no-op. |
| `MEMORY_WORKSPACE_ALLOWLIST` | `""` | Comma-separated workspace ids; empty = every workspace when enabled. |
| `MEMORY_EXTRACT_MODEL` | `claude-haiku-4-5-20251001` | Extraction model. Extraction primarily runs on the conversation agent's own model config (BYOK); `MEMORY_EXTRACT_PROVIDER` / `_API_KEY` / `_BASE_URL` are the fallback when the agent has none pinned. |
| `MEMORY_EXTRACT_DEBOUNCE_S` | `120` | Quiet period before a conversation is extracted. |
| `MEMORY_TOP_K` | `6` | Max memories injected per turn. |
| `MEMORY_MIN_SIMILARITY` | `0.55` | Retrieval floor — below it, nothing is injected (rather than the k least-irrelevant rows). |
| `MEMORY_PROMPT_BUDGET_TOKENS` | `800` | Token budget for the injected block. |
| `MEMORY_RETRIEVAL_TIMEOUT_MS` | `300` | Read-path timeout; on expiry the turn proceeds without memories. |
| `MEMORY_MAX_PER_SUBJECT` | `2000` | Cap per subject; also the export bound. |
| `MEMORY_MAX_PROFILE` | `50` | Cap on workspace-tier (profile) facts per subject. |
| `MEMORY_EMBEDDING_DIM` | `384` | Must match the embedding provider's output dimension. |
