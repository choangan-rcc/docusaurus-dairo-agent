---
sidebar_position: 9
title: Providers
---

# Providers

Endpoints for discovering supported LLM providers and listing the models a
credential can access. These power the model-config creation form in the UI.

## Supported providers

| id | Display name | Kind | Notes |
| --- | --- | --- | --- |
| `anthropic` | Anthropic | native | — |
| `bedrock` | Amazon Bedrock | native | The `base_url` slot carries the **AWS region** (default `us-east-1`); the API key is a composite `ACCESS_KEY_ID:SECRET_ACCESS_KEY`. |
| `gemini` | Google Gemini | native | `google` is accepted as an alias. |
| `openai` | OpenAI | OpenAI-compatible | No base URL → talks to api.openai.com. |
| `groq` | Groq | OpenAI-compatible | Default base URL `https://api.groq.com/openai/v1`. |
| `together` | Together AI | OpenAI-compatible | `https://api.together.xyz/v1` |
| `deepseek` | DeepSeek | OpenAI-compatible | `https://api.deepseek.com/v1` |
| `openrouter` | OpenRouter | OpenAI-compatible | `https://openrouter.ai/api/v1` |
| `xai` | xAI (Grok) | OpenAI-compatible | `https://api.x.ai/v1` |
| `fireworks` | Fireworks AI | OpenAI-compatible | `https://api.fireworks.ai/inference/v1` |
| `ollama` | Ollama (local) | OpenAI-compatible | `http://localhost:11434/v1`; does **not** require an API key. |
| `openai_compatible` | Custom (OpenAI-compatible) | OpenAI-compatible | Escape hatch for vLLM, LM Studio, LiteLLM proxies, etc. **Requires** a `base_url`. |

The offline `fake` provider (used in tests and zero-config setups) is wired into
verification and the runtime but deliberately not advertised in this list.

## `GET /v1/providers`

List advertised providers with form hints for building a credential UI. Returns
a plain array.

```bash
curl -H "Authorization: Bearer $TOKEN" http://localhost:8000/v1/providers
```

**Provider object fields**

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | Provider id (use as `provider` on model configs). |
| `display_name` | string | Human-friendly name. |
| `requires_api_key` | boolean | Whether a key is required (`false` for `ollama`). |
| `requires_base_url` | boolean | Whether a base URL must be supplied (`true` only for `openai_compatible`). |
| `default_base_url` | string \| null | Preset base URL filled in automatically when you omit one. |
| `api_key_label` | string \| null | UI label override (e.g. "AWS credentials" for Bedrock). |
| `api_key_placeholder` | string \| null | UI placeholder override (e.g. `ACCESS_KEY_ID:SECRET_ACCESS_KEY`). |
| `base_url_label` | string \| null | UI label override (e.g. "AWS region" for Bedrock). |

## `POST /v1/providers/{provider}/models`

List the models a credential can access. This is a **POST** (not GET) so the API
key travels in the request body and never appears in URLs or access logs. The
key is used once for this call and never stored.

**Request body**

| Field | Type | Required | Constraints | Description |
| --- | --- | --- | --- | --- |
| `api_key` | string | Yes | ≥ 1 char | The credential to list models with. Used once; never stored or echoed. |
| `base_url` | string \| null | No | ≤ 1024 chars | Custom endpoint (or AWS region for Bedrock). |

```bash
curl -H "Authorization: Bearer $TOKEN" -X POST \
  http://localhost:8000/v1/providers/anthropic/models \
  -H 'Content-Type: application/json' \
  -d '{"api_key": "sk-ant-..."}'
```

**Response** `200`:

```json
{
  "provider": "anthropic",
  "source": "live",
  "items": [
    {
      "id": "claude-sonnet-4-6",
      "display_name": "Claude Sonnet 4.6",
      "context_window": 200000,
      "supports_function_calling": true
    }
  ]
}
```

| Field | Type | Description |
| --- | --- | --- |
| `provider` | string | The provider queried (normalized to lowercase). |
| `source` | string | `live` (fetched from the provider) or `static` (bundled fallback list). |
| `items[].id` | string | Model id. |
| `items[].display_name` | string \| null | Human-friendly name, when known. |
| `items[].context_window` | integer \| null | Context window, when known. |
| `items[].supports_function_calling` | boolean \| null | Tool-calling capability; `null` means unknown. |

**Behavior**

- **Live-first, static fallback**: if the provider is unreachable, the response
  falls back to a static model list (`source: "static"`) where one is bundled
  (Anthropic, Bedrock, OpenAI, and a small Gemini fallback).
- **An invalid credential never falls back** — auth errors are always
  `422 credential_invalid`.
- OpenAI-compatible providers list via `GET /models` with non-chat models
  (embeddings, whisper, TTS, image, moderation, …) filtered out. Gateways other
  than OpenAI itself return bare model ids, and an unreachable host is
  `502 provider_unavailable`.
- Bedrock lists foundation models, keeping only ACTIVE, text-output models;
  models available only through inference profiles are listed under their
  cross-region profile id (`us.` / `eu.` / `apac.` prefix).
- Unknown provider → `422 provider_unsupported`.
- Provider SDK error strings are sanitized — never echoed back in responses.
