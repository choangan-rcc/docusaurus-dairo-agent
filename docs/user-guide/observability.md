---
sidebar_position: 12
title: Observability
---

# Observability

The platform traces agent runs to a self-hosted
[Arize Phoenix](https://arize.com/docs/phoenix) instance, and surfaces those
traces directly in the web UI — so you can see exactly what an agent did during
a conversation: every LLM call, tool invocation, and retrieval, with timings,
token counts, and errors.

Tracing is **off by default** — nothing is exported until you opt in.

## Turning it on

```bash
docker compose up -d phoenix     # start Phoenix locally (part of make infra-up)
```

Then in `.env`:

```bash
PHOENIX_ENABLED=true
PHOENIX_COLLECTOR_ENDPOINT=http://localhost:6006
```

The Phoenix UI itself is at `http://localhost:6006`. In production, run Phoenix
backed by Postgres with authentication enabled, create a system API key in the
Phoenix UI (Settings → API Keys), and set `PHOENIX_API_KEY` on the app — see
[Deployment](/docs/developer-guide/deployment) for the production checklist.

## What gets traced

Each chat turn opens an `agent.turn` root span carrying the agent, pattern,
model, channel, session, and (when routed) routing decision — plus token counts
and estimated cost. Nested under it:

- **LLM spans** — every model call, with inputs and outputs.
- **Tool spans** — each tool invocation with its arguments and result.
- **Retrieval spans** — knowledge-base searches, plus ingestion-side spans
  (`kb.ingest`, `kb.embed`, `kb.index`) for document processing.

Only non-secret identifiers are ever attached to spans — never API keys.

## The Traces page

The **Observability** page in the UI lists recent traces with filters for
agent, time range (1h / 24h / 7d), and errored-only, showing status, time,
agent, model, channel, latency, and tokens per trace. It refreshes
automatically every 30 seconds.

Clicking a trace opens the **trace detail** view: a span tree with per-span
latency bars on the left, and the selected span's inputs, outputs, and
attributes on the right — tool spans render just like tool cards in the
Playground. Each trace also deep-links into the Phoenix UI for deeper analysis.

If tracing is disabled or Phoenix is unreachable, the page degrades to a status
panel explaining the state instead of raw errors.

## Configuration

| Env var | Default | Purpose |
| --- | --- | --- |
| `PHOENIX_ENABLED` | `false` | Master opt-in flag for all tracing. |
| `PHOENIX_COLLECTOR_ENDPOINT` | `http://localhost:6006` | Phoenix base URL (UI + OTLP HTTP collector). |
| `PHOENIX_API_KEY` | — | System API key (required when Phoenix auth is on). |
| `PHOENIX_PROJECT_NAME` | `agentic-platform` | Phoenix project traces are written to. |

A tracing failure never breaks the app: setup errors are swallowed at startup,
and all tracing helpers are no-ops when disabled.
