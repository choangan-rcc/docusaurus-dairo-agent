---
sidebar_position: 5
title: Chat / Playground
---

# Chat / Playground

Run a conversation turn against an agent. Conversations are keyed by a
`session_token` you provide: reuse the same token to continue a conversation
(multi-turn memory); use a new token to start a fresh one. Only the most recent
messages (default 40, `HISTORY_MAX_MESSAGES`) are replayed to the model each
turn — older context is dropped from the prompt, though the full history stays
in the database and the messages endpoint.

## `POST /v1/agents/{agent_id}/playground/chat`

Send one message. Supports **non-streaming** JSON and **SSE streaming**, selected
by the `stream` flag.

**Request body**

| Field | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `session_token` | string | Yes | — | Conversation key. Same token = same conversation. |
| `message` | string | Yes | — | The user message. |
| `stream` | boolean | No | `true` | `true` for SSE streaming, `false` for a single JSON response. |

An unknown agent id fails with `404` **before** any streaming response opens.
If the agent gates any tool behind human approval, non-streaming requests that
would hit a gated tool fail with `400 approval_requires_streaming` — approval
pauses only work over SSE.

### Non-streaming (`"stream": false`)

```bash
curl -H "Authorization: Bearer $TOKEN" -X POST \
  http://localhost:8000/v1/agents/agt_a1b2c3d4/playground/chat \
  -H 'Content-Type: application/json' \
  -d '{ "session_token": "sess_abc123", "message": "What is 21 * 2?", "stream": false }'
```

**Response** `200` (`application/json`):

```json
{
  "message": "21 times 2 is 42.",
  "usage": {
    "tokens_in": 34,
    "tokens_out": 9,
    "latency_ms": 812,
    "cost_usd": 0.00021,
    "model": "claude-sonnet-4-6"
  },
  "conversation_id": "conv_5d6e7f80",
  "message_id": "msg_9a0b1c2d",
  "trace_id": "9f86d081884c7d65…",
  "tool_calls": [
    { "name": "calculator", "kwargs": { "expression": "21 * 2" }, "output": "42.0" }
  ],
  "routing": null,
  "stop_reason": null
}
```

**Response fields**

| Field | Type | Description |
| --- | --- | --- |
| `message` | string | The assistant's reply text. |
| `usage` | object | `{ tokens_in, tokens_out, latency_ms, cost_usd, model }`. |
| `conversation_id` | string | Conversation id (prefix `conv_`); stable per `session_token`. |
| `message_id` | string \| null | Id of the stored assistant message. |
| `trace_id` | string \| null | Phoenix trace id for this turn (when tracing is enabled). |
| `tool_calls` | array | Tools invoked during the turn: `{ name, kwargs, output }`. |
| `routing` | object \| null | Routing decision when the agent uses a [model router](/docs/api-reference/model-routers): `{ router_id, rule_id, rule_name, source, model_config_id, model }` with `source` = `rule` \| `sticky` \| `fallback`. `null` for pinned-model agents. |
| `stop_reason` | string \| null | `"guardrail"` marks a turn blocked by guardrails. |

### Streaming (`"stream": true`, the default)

Returns `text/event-stream`. Each event is a `data:` line whose JSON payload
carries a `type`:

| Event | Payload | Meaning |
| --- | --- | --- |
| `text-delta` | `{"type": "text-delta", "delta": "<text>"}` | Incremental output text — concatenate the deltas. |
| `reasoning-delta` | `{"type": "reasoning-delta", "delta": "<text>"}` | Incremental model reasoning, when the model exposes it. |
| `tool-start` | `{"type": "tool-start", "id", "name", "kwargs"}` | A tool call began. |
| `tool-result` | `{"type": "tool-result", "id", "name", "output"}` | The tool call finished. |
| `approval-required` | `{"type": "approval-required", "approval_id", "tool_name", "tool_kwargs"}` | **Terminal for a paused turn** — a gated tool needs human approval; the stream closes. Resolve via the approvals endpoint below. |
| `done` | `{"type": "done", "done": true, "usage": {...}, "conversation_id", "message_id", "trace_id", "routing", "stop_reason"}` | Terminal success event. |
| `error` | `{"type": "error", "error": {"code", "message"}}` | Terminal error event — runtime errors arrive in-stream because the `200` is already committed. |

```bash
curl -H "Authorization: Bearer $TOKEN" -N -X POST \
  http://localhost:8000/v1/agents/agt_a1b2c3d4/playground/chat \
  -H 'Content-Type: application/json' \
  -d '{ "session_token": "sess_abc123", "message": "Tell me a short greeting." }'
```

```text
data: {"type": "text-delta", "delta": "Hel"}

data: {"type": "text-delta", "delta": "lo!"}

data: {"type": "done", "done": true, "usage": {"tokens_in": 18, "tokens_out": 3, "latency_ms": 402, "cost_usd": 0.00007, "model": "claude-sonnet-4-6"}, "conversation_id": "conv_5d6e7f80", "message_id": "msg_9a0b1c2d", "trace_id": null, "routing": null, "stop_reason": null}
```

## Human-in-the-loop approvals

When a gated tool is about to run, the turn pauses: the stream emits
`approval-required` and closes, and a pending approval is persisted server-side
(durable across disconnects) with a **15-minute TTL**.

A tool is gated by whichever object owns it:

| Tool kind | Gate lives on |
| --- | --- |
| Built-in tools | `approval_tool_names` on the [agent](/docs/api-reference/agents). |
| MCP server tools | `approval_tool_names` on the [MCP server](/docs/api-reference/mcp-servers#approval-gating). |
| Data source tools | `approval_tool_names` on the [data source](/docs/api-reference/data-sources#approval-gating). |

### `POST /v1/agents/{agent_id}/playground/approvals/{approval_id}`

Resolve a pending approval. The response is a **new SSE stream** of the resumed
turn, using the same event vocabulary as the chat stream.

**Request body**

| Field | Type | Required | Constraints | Description |
| --- | --- | --- | --- | --- |
| `approved` | boolean | Yes | — | Approve or deny the tool call. |
| `reason` | string \| null | No | ≤ 2000 chars | Optional reason, shown to the model on denial. |

On **denial**, the tool is not run — the model receives
`"User denied this tool call."` (plus the reason) and continues the turn,
adapting its answer.

**Pre-stream errors** (plain HTTP):

- `404 not_found` — unknown approval (or another workspace's).
- `409 approval_already_resolved` — a decision was already recorded.
- `410 approval_expired` — the 15-minute TTL lapsed.
- `500 approval_resume_failed` — the approved tool call could not be resumed;
  resend the original message.

## Conversations

### `GET /v1/agents/{agent_id}/conversations`

Paginated list of an agent's conversations, most-recent first.

**Conversation object fields**

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | Conversation id (prefix `conv_`). |
| `agent_id` | string | Owning agent id. |
| `session_token` | string | The client-provided conversation key. |
| `title` | string \| null | Optional display title. |
| `channel` | string | `playground` or `agent` (API-driven turns). |
| `message_count` | integer | Total messages (user + assistant). |
| `memory_read_enabled` | boolean | Whether turns in this conversation may read [long-term memory](/docs/api-reference/memories). Default `true`. |
| `memory_write_enabled` | boolean | Whether this conversation is learned from. Default `true`. |
| `created_at` / `updated_at` | string | ISO-8601 timestamps. |

### `GET /v1/agents/{agent_id}/conversations/{conversation_id}`

Fetch a single conversation. Unknown ids — or a conversation that belongs to a
different agent — return `404`.

### `PATCH /v1/agents/{agent_id}/conversations/{conversation_id}`

Rename a conversation: `{ "title": "Billing question" }` (≤ 255 chars;
`"title": null` clears it). Memory switches are **not** set here — they have
their own endpoints below.

## Memory toggles

Two per-conversation switches over [long-term memory](/docs/api-reference/memories).
Both return the same shape:

```json
{
  "conversation_id": "conv_5d6e7f80",
  "memory_read_enabled": true,
  "memory_write_enabled": false
}
```

### `PUT /v1/agents/{agent_id}/conversations/{conversation_id}/memory/read`

**Request body:** `{ "enabled": false }`

Off means this conversation's turns answer without reading memory. Learning is
unaffected and nothing is deleted.

### `PUT /v1/agents/{agent_id}/conversations/{conversation_id}/memory/write`

**Request body:** `{ "enabled": false }`

Off means this conversation is not learned from. Turning it back **on**
fast-forwards the conversation's extraction watermark — anything said while it was
off (including a burst still inside the debounce window) is never extracted, and
learning resumes from now.

:::note
These are the conversation-level switches. Memory also has to be allowed by the
platform flags, the subject's own switches, and the agent's `memory_config` — see
[Memories](/docs/api-reference/memories#per-conversation-and-per-agent-controls).
:::

### `DELETE /v1/agents/{agent_id}/conversations/{conversation_id}`

Delete a conversation and all of its messages. Irreversible. **Response** `204`.

### `GET /v1/agents/{agent_id}/conversations/{conversation_id}/messages`

Paginated message history.

**Message object fields**

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | Message id (prefix `msg_`). |
| `conversation_id` | string | Owning conversation. |
| `role` | string | `user`, `assistant`, or `system`. |
| `content` | string | Message text. |
| `model` | string \| null | Model that produced the message (assistant), or `null`. |
| `trace` | object \| null | Telemetry: tokens, latency, model, stop reason, tool calls, routing, and — when memory was injected — a `memories` array of the memory ids used, in injection order. Resolve them with [`GET /v1/memories/me?ids=…`](/docs/api-reference/memories). |
| `created_at` | string | ISO-8601 timestamp. |
