---
sidebar_position: 5
title: Knowledge Bases
---

# Knowledge Bases

Knowledge bases give agents Retrieval-Augmented Generation (RAG): you upload
documents, the platform parses, chunks, embeds, and indexes them, and any agent
you attach the knowledge base to can search that content while answering.

## Creating a knowledge base

From the **Knowledge Bases** page, create a knowledge base with a name and an
optional description. A new knowledge base starts `empty`; it becomes `ready`
once its first document finishes ingesting.

## Adding documents

Open a knowledge base and use the **Documents** tab:

- **Upload files** — drag-and-drop or pick files. Supported formats: **PDF,
  DOCX, TXT, Markdown, CSV** (up to 20 MB per file by default).
- **Paste text** — the text sub-tab lets you paste raw text with a filename.

Ingestion is asynchronous. Each document moves through
`queued → processing → ready` (or `failed`), and the document list refreshes
automatically every couple of seconds while anything is in flight. A failed
document shows its error reason and attempt count, and can be **retried** with
the re-ingest action. Deleting a document removes its content from the index in
the background.

Each knowledge base holds up to 1000 documents by default.

## Testing retrieval

The **Search** tab runs a retrieval query directly against the knowledge base —
type a question, choose how many results to return (top-k, default 5), and
inspect the ranked chunks with their source files. This is the quickest way to
sanity-check that your documents were ingested well before wiring the KB into an
agent.

## Attaching to agents

Attach a knowledge base to an agent from the agent form's Knowledge Bases
section (or from the KB's own page, which also lists the agents currently using
it). One knowledge base can serve many agents, and one agent can search several
knowledge bases.

At runtime the agent gets a single `search_knowledge_base` tool that knows about
all of its attached KBs — the query is searched across all of them at once and
the best results (top 5 by default, across all KBs) are returned to the model,
each labeled with its source file and knowledge base. A retrieval failure never
crashes the conversation; the model just learns the search didn't work.

## Backends

Out of the box, everything runs locally with zero setup: documents in local
file storage, vectors in embedded Chroma, and deterministic offline embeddings
(fine for trying the feature, but not semantically meaningful). For real
deployments, switch via environment variables to:

- **Embeddings:** OpenAI, Gemini, or Bedrock (`EMBEDDING_PROVIDER`,
  `EMBEDDING_MODEL`, `EMBEDDING_API_KEY`).
- **Vector store:** a Chroma server or Chroma Cloud (`VECTOR_DB_PROVIDER`,
  `CHROMA_SERVER_URL`).
- **File storage:** S3/MinIO (`STORAGE_PROVIDER=s3`, `S3_ENDPOINT`, …).
- **Ingestion queue:** a Redis-backed worker for higher throughput
  (`INGEST_QUEUE=arq` plus `make worker`).

The full settings table is in the
[Knowledge Bases API reference](/docs/api-reference/knowledge-bases#ingestion-pipeline--configuration).
