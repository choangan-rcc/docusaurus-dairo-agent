---
sidebar_position: 16
title: Error Codes
---

# Error Codes

Every error uses the [error envelope](/docs/api-reference/overview#error-envelope)
`{"error": {"code", "message", "details"}}`. Codes and their HTTP statuses:

## General

| Code | HTTP | When it happens |
| --- | --- | --- |
| `validation_error` | 422 | Request body/query failed validation; bad `pattern_config`; unknown tool; missing/invalid referenced `model_config_id` or `router_id`; unknown/disabled `pattern_id`; `approval_tool_names` not a subset of `tool_names`. |
| `not_found` | 404 | Referenced resource does not exist (or belongs to another workspace). |
| `unauthorized` | 401 | Missing, invalid, or expired credentials. |
| `forbidden` | 403 | Authenticated but not allowed — e.g. `X-Workspace-Id` names a workspace you're not a member of. |
| `conflict` | 409 | Uniqueness conflict — e.g. registering an email that already exists. |
| `invalid_current_password` | 400 | `POST /v1/auth/change-password` with the wrong current password. Deliberately not `401`, which clients read as an expired session. |
| `internal_error` | 500 | Unhandled server error. |
| `not_ready` | 503 | `GET /ready` — database unreachable. |

## Agents & chat runtime

| Code | HTTP | When it happens |
| --- | --- | --- |
| `model_source_conflict` | 422 | An agent tried to pin both a `model_config_id` and a `router_id`. |
| `provider_rate_limited` | 429 | The model provider rate-limited the chat turn. |
| `provider_auth_failed` | 422 | The provider rejected auth during a chat turn. |
| `provider_model_not_found` | 422 | The requested model was not found by the provider. |
| `provider_unavailable` | 502 | The provider was unreachable during a chat turn. |
| `provider_error` | 502 | Other/unclassified provider error during a chat turn. |
| `agent_timeout` | 504 | A chat turn exceeded the pattern's `timeout_seconds`. |
| `agent_engine_error` | 502 | The agent engine failed during a turn. |
| `pattern_unavailable` | 500 | The agent's pattern is no longer in the code registry. |
| `pattern_llm_incompatible` | 422 | E.g. `function_agent` on a model without function calling. |
| `llm_misconfigured` / `llm_provider_unavailable` / `llm_provider_unknown` | 500 | Server-side provider configuration problems (missing key, missing integration package, unknown provider name). |

:::note
During a **streaming** chat turn, runtime errors (`provider_*`,
`agent_timeout`, `pattern_unavailable`, …) arrive as a terminal SSE event
`{"type": "error", "error": {"code", "message"}}` rather than an HTTP status,
because the `200` stream is already open.
:::

## Approvals (human-in-the-loop)

| Code | HTTP | When it happens |
| --- | --- | --- |
| `approval_requires_streaming` | 400 | A gated tool was hit on a non-streaming (`stream: false`) turn. |
| `approval_already_resolved` | 409 | The approval was already approved/denied (or a concurrent decision won). |
| `approval_expired` | 410 | The 15-minute approval TTL lapsed. |
| `approval_resume_failed` | 500 | The approved tool call could not be resumed; resend the message. |

## Credentials & routing

| Code | HTTP | When it happens |
| --- | --- | --- |
| `credential_invalid` | 422 | Model config key rejected by the provider (nothing stored), or an invalid key on the model-listing endpoint. |
| `credential_verification_upstream` | 502 | Could not reach the provider to verify a key. |
| `base_url_required` | 422 | The provider preset requires a `base_url` and none was supplied. |
| `provider_unsupported` | 422 | Unknown provider on the model-listing endpoint. |
| `credential_encryption_misconfigured` | 500 | `CREDENTIAL_ENCRYPTION_KEY` missing or malformed. |
| `credential_decrypt_failed` | 500 | Stored credential could not be decrypted (wrong key / corrupted data). |
| `router_capability_unsupported` | 422 | A router target model is known not to support function calling. |
| `router_fallback_unavailable` | 422 | The router's fallback model config is missing; repair the router. |

## MCP servers

| Code | HTTP | When it happens |
| --- | --- | --- |
| `mcp_discovery_failed` | 502 | Could not connect to the MCP server during `/discover`. |
| `mcp_unknown_tools` | 422 | Attach request named tools not in `discovered_tools`; run discovery first. |

## Data sources

| Code | HTTP | When it happens |
| --- | --- | --- |
| `datasource_validation_failed` | 422 | Submitted values don't match the provider descriptor; duplicate data source name; approval/attach request named tools not in `discovered_tools`. |
| `datasource_auth_failed` | 422 | The backing system rejected the supplied credentials. |
| `datasource_unreachable` | 422 | Spawn/connect/timeout failure before the credentials were judged. |
| `datasource_disabled` | 409 | Operation attempted on a disabled or disconnected source. |
| `datasource_provider_unknown` | 500 | A stored row (or request) names a provider this build no longer ships. |

:::note
A **failed connection probe is not an error response** — `POST /v1/data-sources`
and `/test` return `200`/`201` with `status: "failed"` and a fixed
`status_reason`, so the connection can be repaired without retyping secrets.
Provider/driver text is never forwarded, only classified into one of five
phrases. See [Data Sources](/docs/api-reference/data-sources#statuses).
:::

## Knowledge bases

| Code | HTTP | When it happens |
| --- | --- | --- |
| `kb_quota_exceeded` | 429 | The KB already holds the per-KB document quota (default 1000). |
| `kb_upload_unsupported_type` | 415 | File extension not in `.pdf`, `.docx`, `.txt`, `.md`, `.csv`. |
| `kb_upload_too_large` | 413 | Upload exceeds the size cap (default 20 MB). |
| `kb_document_busy` | 409 | Re-ingest requested while the document isn't in a terminal state. |
| `kb_search_failed` | 502 | Vector/embedding backend failed during a search. |

## Observability & usage

| Code | HTTP | When it happens |
| --- | --- | --- |
| `tracing_disabled` | 409 | Trace endpoints called while `PHOENIX_ENABLED=false`. |
| `tracing_unavailable` | 502 | Phoenix unreachable or errored. |
| `invalid_days` | 422 | `GET /v1/usage/overview` with `days` not in `7`, `30`, `90`. |
