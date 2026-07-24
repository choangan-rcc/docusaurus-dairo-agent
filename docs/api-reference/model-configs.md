---
sidebar_position: 8
title: Model Configs (BYOK)
---

# Model Configs (BYOK)

A **model config** bundles a `provider` + `model` + `api_key` (bring your own
key). The key is **verified with a real provider call before being stored** — a
bad key is rejected with `422` and nothing is persisted. Keys are stored
encrypted with AES-256-GCM; the raw key is never returned or accepted back —
responses contain only a masked `key_hint` (the last 4 characters).

Providers are normalized to lowercase on store. See
[Providers](/docs/api-reference/providers) for the full list of supported
providers (Anthropic, Bedrock, Gemini, OpenAI, and a family of OpenAI-compatible
gateways such as Groq, Together, DeepSeek, OpenRouter, xAI, Fireworks, Ollama,
and a custom `openai_compatible` escape hatch). The offline `fake` provider
accepts any key.

There is no update endpoint — keys are write-only, so replacing a key means
delete + recreate.

Deleting a model config nulls out `model_config_id` on any agent that references
it, so revocation never breaks a running agent — it falls back to the
environment-configured provider. Model routers that reference the config are
left intact; dangling targets are detected at read/turn time.

## `POST /v1/model-configs`

Create (verify + store) a model config. Returns the masked view.

**Request body**

| Field | Type | Required | Default | Constraints | Description |
| --- | --- | --- | --- | --- | --- |
| `name` | string | Yes | — | 1–255 chars | Human-friendly label. |
| `provider` | string | Yes | — | 1–64 chars | Provider name. Lowercased on store. |
| `model` | string | Yes | — | 1–128 chars | Provider model id (e.g. `claude-sonnet-4-6`). |
| `api_key` | string | Yes | — | ≥ 1 char | The API key. Write-only; never returned. For Bedrock this is a composite `ACCESS_KEY_ID:SECRET_ACCESS_KEY`. |
| `base_url` | string \| null | No | `null` | ≤ 1024 chars | Custom endpoint base URL. When omitted, the provider preset's default base URL is filled in automatically. For Bedrock this slot carries the **AWS region**. |

```bash
curl -H "Authorization: Bearer $TOKEN" -X POST \
  http://localhost:8000/v1/model-configs \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "Prod Anthropic key",
    "provider": "anthropic",
    "model": "claude-sonnet-4-6",
    "api_key": "sk-ant-abc123def4"
  }'
```

**Response** `201`:

```json
{
  "id": "mcfg_7f3a9c21",
  "name": "Prod Anthropic key",
  "provider": "anthropic",
  "model": "claude-sonnet-4-6",
  "key_hint": "…def4",
  "base_url": null,
  "key_version": 1,
  "created_at": "2026-07-14T09:05:00Z",
  "last_used_at": null
}
```

**Model config object fields**

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | Model config id (prefix `mcfg_`). |
| `name` | string | The label you provided. |
| `provider` | string | Provider name (lowercased). |
| `model` | string | Provider model id. |
| `key_hint` | string | Masked key: `…` plus the last 4 characters. Never the full key. |
| `base_url` | string \| null | Custom endpoint (or preset default), if set. |
| `key_version` | integer | Encryption key version used to store the credential. |
| `created_at` | string | ISO-8601 creation timestamp. |
| `last_used_at` | string \| null | ISO-8601 timestamp of last runtime use, or `null`. Touched each time the key is decrypted for a chat turn. |

**Errors**

- `422 credential_invalid` — the provider rejected the key (or the provider is
  unsupported for verification). **Nothing is stored.**
- `502 credential_verification_upstream` — the provider could not be reached to
  verify the key.
- `422 base_url_required` — the provider preset requires a base URL (the custom
  `openai_compatible` provider) and none was supplied.

## `GET /v1/model-configs`

List model configs (masked), most-recent first. Paginated — see
[Pagination](/docs/api-reference/overview#pagination).

```bash
curl -H "Authorization: Bearer $TOKEN" 'http://localhost:8000/v1/model-configs?limit=20&offset=0'
```

**Response** `200` — a page of model config objects.

## `GET /v1/model-configs/{model_config_id}`

Fetch a single model config (masked).

```bash
curl -H "Authorization: Bearer $TOKEN" http://localhost:8000/v1/model-configs/mcfg_7f3a9c21
```

**Response** `200` — a single model config object. Unknown id returns `404`.

## `DELETE /v1/model-configs/{model_config_id}`

Delete a model config. Any agent referencing it has its `model_config_id` set to
`null` (falls back to env keys). Model routers referencing it are left intact —
because a `204` response has no body, the number of routers still referencing
the deleted config is returned in an **`X-Router-References`** response header so
clients can warn the user.

```bash
curl -H "Authorization: Bearer $TOKEN" -X DELETE \
  http://localhost:8000/v1/model-configs/mcfg_7f3a9c21
```

**Response** `204` — no body. Unknown id returns `404`.

## Key encryption

BYOK keys are encrypted with **AES-256-GCM** before storage. The master key
comes from the `CREDENTIAL_ENCRYPTION_KEY` environment variable — a
base64-encoded 32-byte key — and each credential row records the `key_version`
used, so key rotation is additive. Decryption happens lazily at chat-turn time;
each use touches `last_used_at` and emits an audit-log line (config id, provider,
workspace, timestamp — never key material).

Misconfiguration surfaces as `500 credential_encryption_misconfigured`; a failed
decryption (wrong key, corrupted data) as `500 credential_decrypt_failed`.
