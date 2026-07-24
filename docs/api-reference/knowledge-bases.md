---
sidebar_position: 12
title: Knowledge Bases
---

# Knowledge Bases

A **knowledge base** (KB) holds ingested documents that agents can search at
runtime (Retrieval-Augmented Generation). Documents are parsed, chunked,
embedded, and indexed into a per-KB vector collection; agents attached to one or
more KBs get a `search_knowledge_base` tool automatically.

All routes require admin auth and are workspace-scoped.

## Knowledge base object (`KnowledgeBaseOut`)

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | KB id (prefix `kb_`). |
| `name` | string | KB name. |
| `description` | string \| null | Optional description. |
| `strategy` | string | Retrieval strategy (currently `vector`). |
| `status` | string | `empty` (no ingested documents) or `ready`. |
| `created_at` / `updated_at` | string | ISO-8601 timestamps. |

## `POST /v1/knowledge-bases`

Create a knowledge base.

**Request body**

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `name` | string | Yes | 1–255 chars |
| `description` | string \| null | No | |

```bash
curl -H "Authorization: Bearer $TOKEN" -X POST \
  http://localhost:8000/v1/knowledge-bases \
  -H 'Content-Type: application/json' \
  -d '{"name": "Product docs", "description": "Public documentation"}'
```

**Response** `201` — the KB object with `status: "empty"`.

## `GET /v1/knowledge-bases`

List knowledge bases. Paginated — see
[Pagination](/docs/api-reference/overview#pagination).

## `GET /v1/knowledge-bases/{kb_id}`

Fetch a single KB. Unknown id returns `404`.

## `PATCH /v1/knowledge-bases/{kb_id}`

Update `name` and/or `description` (PATCH semantics).

## `DELETE /v1/knowledge-bases/{kb_id}`

Delete a KB. Clears its vector collection, deletes each document's stored
object, then removes document rows, agent-attachment links, and the KB row.
Backend cleanup is best-effort — a failing vector/storage backend never blocks
the deletion.

**Response** `204`.

## Documents

### Document object (`KbDocumentOut`)

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | Document id (prefix `doc_`). |
| `kb_id` | string | Owning KB. |
| `filename` | string | Original filename. |
| `content_type` | string \| null | MIME type. |
| `size_bytes` | integer \| null | Stored size. |
| `status` | string | `queued` → `processing` → `ready` \| `failed`, plus `deleting` (tombstone during async cleanup). |
| `error_reason` | string \| null | Failure reason when `status: "failed"`. |
| `attempts` | integer | Ingestion attempts so far. |
| `created_at` | string | ISO-8601 timestamp. |

### `POST /v1/knowledge-bases/{kb_id}/documents`

Ingest raw text as a document. Returns `202 Accepted` with the document in
`status: "queued"` — poll the document endpoint for completion.

**Request body**

| Field | Type | Required | Default | Constraints |
| --- | --- | --- | --- | --- |
| `filename` | string | Yes | — | 1–512 chars |
| `content` | string | Yes | — | ≥ 1 char |
| `content_type` | string | No | `text/plain` | |

**Errors:** `404 not_found`; `429 kb_quota_exceeded` when the KB already holds
the per-KB document quota (default 1000, `KB_MAX_DOCUMENTS`).

### `POST /v1/knowledge-bases/{kb_id}/documents/upload`

Upload a file (multipart form, single `file` field). Returns `202` with a
`queued` document.

Supported extensions: `.pdf`, `.docx`, `.txt`, `.md`, `.csv`.

**Errors:** `415 kb_upload_unsupported_type` (unsupported extension);
`413 kb_upload_too_large` (over the upload cap, default 20 MB via
`MAX_UPLOAD_MB` — enforced while streaming); `429 kb_quota_exceeded`; `404`.

```bash
curl -H "Authorization: Bearer $TOKEN" -X POST \
  http://localhost:8000/v1/knowledge-bases/kb_123/documents/upload \
  -F 'file=@handbook.pdf'
```

### `GET /v1/knowledge-bases/{kb_id}/documents`

List a KB's documents. Returns a **plain array** (not paginated).

### `GET /v1/knowledge-bases/{kb_id}/documents/{doc_id}`

Fetch a single document — the status-polling endpoint. `404` if the document is
missing or belongs to another KB.

### `POST /v1/knowledge-bases/{kb_id}/documents/{doc_id}/reingest`

Re-run ingestion for a document. Only valid from `ready` or `failed` (resets
`attempts` to 0 and clears `error_reason`); otherwise `409 kb_document_busy`.

**Response** `202` — the document back in `status: "queued"`.

### `DELETE /v1/knowledge-bases/{kb_id}/documents/{doc_id}`

Delete a document. Two-phase: the row is marked `deleting` synchronously, then a
background job clears its vectors and stored object and removes the row.

**Response** `204`.

## Search

### `POST /v1/knowledge-bases/{kb_id}/search`

Run a retrieval query directly against one KB (useful for testing retrieval
quality without an agent).

**Request body**

| Field | Type | Required | Default | Constraints |
| --- | --- | --- | --- | --- |
| `query` | string | Yes | — | ≥ 1 char |
| `top_k` | integer | No | `5` | `1 ≤ x ≤ 50` |

**Response** `200` — an array of results:

```json
[
  {
    "text": "…chunk text…",
    "score": 0.83,
    "metadata": { "filename": "handbook.pdf", "kb_id": "kb_123" }
  }
]
```

**Errors:** `404`; `502 kb_search_failed` if the vector/embedding backend fails.

## Attaching KBs to agents

### `GET /v1/agents/{agent_id}/knowledge-bases`

List KBs attached to an agent. Returns a plain array of link objects
`{ id, kb_id }` (link id prefix `akb_`).

### `POST /v1/agents/{agent_id}/knowledge-bases`

Attach a KB to an agent. **Idempotent** — re-attaching returns the existing link
(still `201`).

**Request body:** `{ "kb_id": "kb_123" }`

### `DELETE /v1/agents/{agent_id}/knowledge-bases/{kb_id}`

Detach a KB from an agent. `204` on success; `404` if the link doesn't exist.

### `GET /v1/knowledge-bases/{kb_id}/agents`

Reverse lookup: list the agents using this KB, as `{ id, name, status }`
objects.

## Runtime behavior

When an agent has at least one attached KB, the engine adds a single
`search_knowledge_base(query)` tool whose description enumerates the attached
KBs by name. At call time the query is embedded once and fanned out across all
attached KBs concurrently; results are merged, filtered by
`KB_RETRIEVAL_MIN_SCORE` (default 0 = off), sorted by score, and truncated to
`KB_RETRIEVAL_TOP_K` (default 5) **total across all KBs**. Each hit is prefixed
with its source: `[source: <filename> | knowledge base: <kb name>]`. Retrieval
failures return an error string to the model — they never crash the turn.

## Ingestion pipeline & configuration

Documents flow through **parse → chunk → embed → index**:

- **Parse:** `.txt`/`.md`/`.csv` decoded as UTF-8; `.pdf` via pypdf; `.docx` via
  docx2txt.
- **Chunk:** sentence splitting with chunk size 512 and overlap 64.
- **Embed:** the configured embeddings backend.
- **Index:** the KB's own vector collection (collection name = KB id), with
  idempotent re-indexing per document.

The queue is in-process by default; set `INGEST_QUEUE=arq` to move ingestion to
an arq/Redis worker (`make worker`). A stale-job sweep requeues `processing`
jobs older than `INGEST_STALE_AFTER_SECONDS` (default 900) up to
`INGEST_MAX_ATTEMPTS` (default 3), then marks them `failed`.

**Key settings** (env vars):

| Setting | Default | Purpose |
| --- | --- | --- |
| `EMBEDDING_PROVIDER` | `fake` | `fake` \| `openai` \| `gemini` \| `bedrock` |
| `EMBEDDING_MODEL` / `EMBEDDING_API_KEY` / `EMBEDDING_BASE_URL` / `EMBEDDING_REGION` | — | Provider-specific embedding settings. |
| `EMBEDDING_BATCH_SIZE` | `100` | Chunks per embedding request. |
| `VECTOR_DB_PROVIDER` | `chroma` | `chroma` \| `chroma_cloud` (OpenSearch support is in progress). |
| `VECTOR_DB_DATA_DIR` | `./knowledge_bases` | Embedded Chroma persistence dir. |
| `CHROMA_SERVER_URL` | — | Use a Chroma server instead of embedded mode. |
| `KB_RETRIEVAL_TOP_K` | `5` | Merged top-k for the agent retrieval tool. |
| `KB_RETRIEVAL_MIN_SCORE` | `0.0` | Score floor for retrieval hits (0 = disabled). |
| `STORAGE_PROVIDER` | `local` | `local` \| `s3` (MinIO-compatible). |
| `S3_ENDPOINT` / `S3_BUCKET` / `S3_ACCESS_KEY` / `S3_SECRET_KEY` | `S3_BUCKET=agentic-kb` | S3/MinIO settings. |
| `INGEST_QUEUE` | `inprocess` | `inprocess` \| `arq`. |
| `REDIS_URL` | `redis://localhost:6379` | arq queue backend. |
| `MAX_UPLOAD_MB` | `20` | Upload size cap. |
| `KB_MAX_DOCUMENTS` | `1000` | Per-KB document quota. |
