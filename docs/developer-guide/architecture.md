---
sidebar_position: 1
title: Backend Architecture
---

# Backend Architecture

The backend is a Python/FastAPI application that lives in `src/agentic_platform/` and
is managed with [`uv`](https://docs.astral.sh/uv/). This page describes how the
application is put together and the conventions that every subsystem follows.

## Application startup

The FastAPI app is created by an **app factory** in `main.py`. Its lifespan handler
runs the platform's startup sequence:

1. Set up **Phoenix tracing** (this must happen before any LlamaIndex engine is
   built, so instrumentation hooks are in place first).
2. Initialize the database.
3. Seed the default workspace.
4. Sync the **pattern catalog** from code into the database.
5. When the in-process ingestion queue is used, sweep stranded ingestion jobs left
   behind by a previous process.

Configuration is entirely env-driven via `config.py`, which defines a
pydantic-settings `Settings` class exposed through an `lru_cache`'d
`get_settings()`. `config.py` is treated as a fixed contract — its import order is
deliberately excluded from ruff's isort rule.

The platform runs with **zero-configuration defaults**: SQLite database, the `fake`
LLM provider (no API key), an in-process ingestion queue, local file storage, and
Chroma as the vector store. Every one of those defaults can be swapped for a
production-grade backend (Postgres + pgvector, arq/Redis, S3/MinIO, OpenSearch)
through environment variables alone.

## Key conventions

### Workspace-scoped repository layer

Feature code **never queries models directly**. All database access goes through the
repository layer in `database/repository.py`. A repository is constructed with a
`workspace_id`, and every query it issues filters on that `workspace_id` — this is
the multi-tenancy insurance, not a convention.

Feature routes depend on `WorkspaceId` from `api/deps.py`: the caller's active
workspace, resolved from the bearer token plus the `X-Workspace-Id` header
validated against membership (falling back to the user's earliest membership).
Because scoping is mandatory in the repository, a cross-tenant id surfaces as a
`404`, never a `403`.

### Single error envelope

Every error response — validation, not-found, auth, provider failures, unexpected
server errors — uses one consistent shape, produced by `api/errors.py`:

```json
{ "error": { "code": "not_found", "message": "...", "details": null } }
```

See the [API Reference](/docs/api-reference/overview) for the full error-code table.

### Authentication

`/v1` routes require a **JWT bearer token** (`security/tokens.py`, with
server-side session rows so logout and revocation take effect). The auth flows
themselves and `/health` / `/ready` are unauthenticated; the latter two are
mounted at the root. A shared-credential HTTP Basic mode survives as a legacy
escape hatch behind `ENABLE_LEGACY_BASIC_AUTH`, disabled by default — its
principal is synthetic, workspace-blind, and deliberately has no memory subject.

### The registry idiom

LLM providers, agent patterns, tools, embeddings backends, vector databases, and
storage backends each **self-register** via a `@register_*` decorator. Adding an
implementation is one new module plus an import in the package's `__init__.py` —
no changes to existing implementations.

Two rules keep this safe:

- Class paths are never stored in the database.
- Nothing is ever imported dynamically from database data.

The database only ever stores stable string identifiers (`pattern_id`, provider
names, backend names) that are looked up in the in-code registries.

### Fake implementations for offline CI

The platform ships a `fake` LLM provider and `fake` embeddings backend. The entire
test suite must pass with **no network access and no API keys** — CI runs against
in-memory SQLite and the fake providers.

## Agent runtime (`agents/`) — hexagonal

The agent runtime follows a hexagonal (ports-and-adapters) layout:

- **`domain/`** — engine-agnostic entities (`AgentTurnResult`, `ToolTrace`) and
  ports: the `AgentEngine` interface, `EngineRequest`, and streamed `EngineEvent`s
  — including `ApprovalRequired`, which powers human-in-the-loop tool approval.
- **`application/`** — the `AgentService` use case. A chat turn flows through
  guardrails → engine → persistence → usage capture, all through org-scoped
  repositories.
- **`infrastructure/`** — adapters: the LlamaIndex engine, the multi-provider LLM
  registry (fake / anthropic / openai / gemini / bedrock), built-in tools, MCP tool
  adapters, and knowledge-base retrieval tools.
- **`patterns/`** — named, versioned recipes (`function_agent`, `react_agent`) that
  map a `pattern_id` to executable code. The `agent_patterns` database table is
  only the display/enable catalog — the code registry is the source of truth.

Every pattern config inherits `PatternSafetyConfig` (max iterations, timeout) with
`extra="ignore"`, so agents created against an older pattern version keep working
after a pattern upgrade — unknown config fields are dropped, not rejected.

:::note
Always import from `agentic_platform.agents` (the package), not its submodules —
the internal layout may evolve.
:::

## Other subsystems

| Package | Responsibility |
| --- | --- |
| `llm/` | Provider catalog, BYOK key verification, model resolution, and intelligent routing (model routers pick a model per request). |
| `ingestion/` | Knowledge-base document pipeline (parse → chunk → embed → index) **and the arq worker** (`ingestion/worker.py`), which also hosts the memory tasks. Queue is in-process by default, or arq/Redis with `INGEST_QUEUE=arq`; under arq the worker owns the stale-job sweep and the nightly memory crons. |
| `datasources/` | Data-source connectors: the curated provider catalog (`templates.py`, registered via the registry idiom), the AES-GCM credential vault, template rendering, and live connection validation. Ships **no tool code** — every provider is a third-party MCP server launched from a version-pinned template. See [Data Sources](/docs/api-reference/data-sources). |
| `memory/` | Long-term memory. `retrieval.py` is the synchronous, fail-open read path; `pipeline.py` the async extract → embed → consolidate write path; plus `store.py` (mandatory `(workspace, subject)` filters), `scope.py` (category → scope whitelist, assigned by code and never by the LLM), `gates.py` (feature gates shared by both paths), `prompts.py`, and `queue.py`. See [Memories](/docs/api-reference/memories). |
| `vectordb/` | Pluggable vector stores (Chroma, OpenSearch), selected via settings. |
| `embeddings/` | Pluggable embeddings backends (including `fake` for CI). |
| `storage/` | Pluggable blob storage (local filesystem, S3/MinIO). |
| `security/crypto.py` | AES-256-GCM encryption for stored BYOK keys (`CREDENTIAL_ENCRYPTION_KEY`). API responses only ever expose a masked `key_hint`. |
| `observability.py` | Arize Phoenix tracing, gated by `PHOENIX_ENABLED` (off by default). `api/observability.py` serves trace data to the frontend. |

## Background worker

With `INGEST_QUEUE=arq`, one worker process
(`arq agentic_platform.ingestion.worker.WorkerSettings`) owns all deferred work:

| Task | Trigger |
| --- | --- |
| `ingest_document` / `delete_document` | Enqueued by the knowledge-base routes. |
| `extract_memories` | Enqueued after a chat turn, debounced per conversation (`MEMORY_EXTRACT_DEBOUNCE_S`). |
| `sweep_stale` | Cron every 5 minutes — re-claims stranded ingestion jobs. |
| `sweep_memory_extractions` | Nightly cron (03:30) — re-enqueues conversations whose extraction watermark lags, which is how a dropped enqueue self-heals. |
| `purge_memory_superseded` | Nightly cron (04:15) — hard-deletes superseded memories older than 90 days. |

The memory **write path only runs under arq**: with the default in-process queue,
extraction is a no-op and `GET /v1/memories/state` honestly reports learning as
unavailable.

## API surface

All feature routes are mounted under `/v1` and wired together in `api/router.py`:
`auth`, `workspaces` (plus members and invites), `agents` (plus templates,
duplicate, versions, and the playground chat with SSE streaming and approval
resolution), `model-configs`, `model-routers`, `patterns`, `mcp-servers`,
`data-sources` and `data-source-providers`, `knowledge-bases`, `memories`
(plus per-agent attachment routers for KBs, MCP servers, and data sources),
`providers/{provider}/models`, `tools`, `guardrails/settings`, `usage/overview`,
and `observability`. Interactive OpenAPI docs are served at `/docs`.

## Tests

The suite uses `pytest` with `asyncio_mode = "auto"` and `pythonpath = ["src"]`.
`tests/conftest.py` gives every test:

- a fresh **in-memory SQLite** database with the pattern catalog pre-synced,
- the `fake` LLM provider force-patched in, and
- a test encryption key.

A developer's local `.env` must never leak into the suite. Because
`get_settings()` is lru-cached, tests patch attributes on the settings singleton
rather than re-instantiating it.

```bash
make test                                   # full suite
uv run pytest tests/test_chat.py -q         # single file
uv run pytest tests/test_chat.py -k name -q # single test
```
