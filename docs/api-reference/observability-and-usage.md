---
sidebar_position: 15
title: Observability & Usage
---

# Observability & Usage

Two read-only surfaces: the **observability** endpoints proxy trace data from a
self-hosted Arize Phoenix instance, and the **usage** endpoint aggregates
per-turn usage events (tokens, cost, latency) recorded by the platform itself.

All routes require admin auth.

## Observability

Tracing is feature-flagged by `PHOENIX_ENABLED` (default **off**). When enabled,
every chat turn opens a root `agent.turn` span (with agent, pattern, model,
channel, session, and routing metadata plus token/cost attributes), and
LLM/tool/retrieval spans nest under it automatically. Knowledge-base ingestion
and search also emit spans (`kb.ingest`, `kb.embed`, `kb.index`, `kb.search`).

The endpoints below never return a raw 500 for Phoenix problems:

- Tracing disabled → `409` with code `tracing_disabled`.
- Phoenix unreachable / errored → `502` with code `tracing_unavailable`.
- The Phoenix API key and collector URL never leak into responses.

### `GET /v1/observability/status`

Never fails. Reports whether tracing is on and whether Phoenix is reachable.

```json
{ "enabled": true, "reachable": true, "phoenix_project": "agentic-platform" }
```

| Field | Type | Description |
| --- | --- | --- |
| `enabled` | boolean | The `PHOENIX_ENABLED` flag. |
| `reachable` | boolean \| null | `null` when disabled (no outbound call); otherwise the result of a ~2s health probe. |
| `phoenix_project` | string | Phoenix project name traces are written to. |

### `GET /v1/observability/traces`

List recent traces (root spans), newest first.

**Query parameters**

| Name | Type | Default | Description |
| --- | --- | --- | --- |
| `agent_id` | string \| null | — | Filter by agent. |
| `errored` | boolean \| null | — | `true` → only errored traces; `false` → only OK. |
| `start_time` / `end_time` | datetime \| null | — | Time window. |
| `limit` | integer | `50` | `1 ≤ x ≤ 200`. No offset pagination. |

**Response** `200` — `{ "data": [TraceSummary] }`:

| Field | Type | Description |
| --- | --- | --- |
| `trace_id` | string | 32-char hex trace id. |
| `root_span_id` | string | Root span id. |
| `name` | string | Root span name (`agent.turn`). |
| `start_time` | datetime \| null | Turn start. |
| `latency_ms` | number \| null | Total trace latency. |
| `status` | string | `ok` or `error`. |
| `agent_id` / `pattern_id` / `model` / `channel` / `session_id` | string \| null | Turn metadata from the root span. |
| `tokens_in` / `tokens_out` | integer \| null | Token counts. |
| `cost_usd` | number \| null | Estimated cost. |

### `GET /v1/observability/traces/{trace_id}`

Fetch a full trace as a flat span list (parent/child via `parent_id`), sorted by
start time. `404` when Phoenix has no spans for the id.

**Response** `200` — `{ "trace_id": "...", "spans": [SpanOut] }`:

| Field | Type | Description |
| --- | --- | --- |
| `span_id` / `parent_id` | string / string \| null | Tree structure. |
| `name` | string | Span name. |
| `span_kind` | string \| null | OpenInference kind (`AGENT`, `LLM`, `TOOL`, `RETRIEVER`, …). |
| `start_time` / `end_time` / `latency_ms` | — | Timing. |
| `status` / `status_message` | string / string \| null | `ok`/`error` plus the error message. |
| `attributes` | object | Span attributes; string values truncated at 4000 chars. |

## Usage

One usage event is recorded per assistant turn: tokens in/out, wall-clock
latency, and an estimated cost computed from a static per-model pricing table
(USD per million tokens; unknown models cost `0`). Deleting an agent keeps its
usage but folds it into a "deleted agents" bucket (`agent_id: null`).

### `GET /v1/usage/overview`

Aggregated usage for the workspace over a trailing window.

**Query parameters**

| Name | Type | Default | Description |
| --- | --- | --- | --- |
| `days` | integer | `30` | One of `7`, `30`, `90`. Anything else → `422` with code `invalid_days`. |

The window is anchored to UTC midnight and spans exactly `days` calendar
buckets, today (partial) included.

**Response** `200`:

```json
{
  "period": { "from": "2026-06-25T00:00:00Z", "to": "2026-07-24T12:00:00Z", "days": 30 },
  "totals": { "requests": 128, "tokens_in": 84210, "tokens_out": 22110, "cost_usd": 1.83, "avg_latency_ms": 2100 },
  "timeseries": [
    { "date": "2026-07-23", "requests": 12, "tokens_in": 8000, "tokens_out": 2100, "cost_usd": 0.21 }
  ],
  "by_agent": [
    { "agent_id": "agt_a1b2c3d4", "agent_name": "Support Bot", "requests": 90, "tokens_in": 60000, "tokens_out": 15000, "cost_usd": 1.20, "avg_latency_ms": 1900 },
    { "agent_id": null, "agent_name": null, "requests": 5, "tokens_in": 2000, "tokens_out": 400, "cost_usd": 0.02, "avg_latency_ms": 1500 }
  ],
  "by_model": [
    { "model": "claude-sonnet-4-6", "requests": 128, "tokens_in": 84210, "tokens_out": 22110, "cost_usd": 1.83 }
  ]
}
```

Notes:

- `timeseries` buckets are UTC days and **sparse** — days with no usage are
  omitted (clients zero-fill).
- `by_agent` and `by_model` are sorted by cost descending and capped at the top
  **50** rows.
- A `by_agent` row with `agent_id: null` is the deleted-agents bucket.
- Zero data returns `200` with all-zero totals and empty arrays — never `404`.
