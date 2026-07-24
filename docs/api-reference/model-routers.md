---
sidebar_position: 10
title: Model Routers
---

# Model Routers

A **model router** bundles an ordered list of routing rules plus a required
fallback model config. An agent points at a router *instead of* a single model
config — `router_id` and `model_config_id` are mutually exclusive on agents
(`422 model_source_conflict`). At the start of each chat turn, the router
evaluates its rules and picks which model config serves that turn.

Routers hold **no credentials** — they reference model configs by id. The
references are deliberately not hard foreign keys, so deleting a model config
never blocks; dangling references are detected at read/turn time and surfaced to
the UI via `exists: false`.

## Routing semantics

Evaluation runs once per turn, before the engine, with precedence
**rules → sticky → fallback**:

1. **Rules, re-run every turn** — rules are evaluated top-to-bottom; the first
   match wins (even over a live sticky entry for a different model). A rule
   that fails to validate or whose target config was deleted is skipped, never
   an error.
2. **Sticky reuse** — only when no rule matched: if the conversation has a live
   sticky entry (within `sticky_window_seconds` of its last use), the previous
   model is reused and the window resets. This prevents model-hopping
   mid-conversation.
3. **Fallback** — the router's `fallback_model_config_id`. Fallback turns are
   never persisted to sticky state, so the next message re-runs the full
   cascade. If the fallback config itself is missing, the turn fails with
   `422 router_fallback_unavailable` — repair the router before turns can route.

Chat responses include a `routing` object describing the decision — see
[Chat / Playground](/docs/api-reference/chat).

## Rule schema

Each rule in `rules` has:

| Field | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `id` | string | No | server-generated | Stable rule id (prefix `rule`). Omit on create; preserved across edits. |
| `name` | string | Yes | — | 1–255 chars. |
| `type` | string | Yes | — | Currently only `calculated`. |
| `model_config_id` | string | Yes | — | Target model config for this rule. |
| `conditions` | object | Yes (for `calculated`) | — | `{ "operator": "AND" \| "OR", "clauses": [...] }` with at least one clause. |

Each clause is a `property` / `comparator` / `value` triple. `value` is always a
**string** on the wire and parsed at evaluation time:

| Property | Comparators | Value semantics |
| --- | --- | --- |
| `prompt_content` | `contains`, `matches`, `eq`, `neq` | `contains` = case-insensitive, comma-separated any-match; `matches` = regex (validated at write time). |
| `conversation_token_count` | `gt`, `gte`, `lt`, `lte`, `eq`, `neq`, `between` | Integer; `between` = `"min,max"` inclusive. Token counts are estimated (`len(text) / 4`). |
| `conversation_message_count` | same numeric set | Integer. |
| `current_hour` | same numeric set | Integers 0–23; `between` with min > max wraps past midnight (e.g. `"22,6"` = 10pm–6am). |
| `has_image_attachment` | `eq` | Boolean string. Accepted but currently inert (chat messages are plain text). |

Invalid comparator/property combinations, unparseable values, out-of-range
hours, and invalid regexes are rejected with `422` at write time.

## Router object (`RouterOut`)

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | Router id (prefix `rtr_`). |
| `name` | string | Router name. |
| `description` | string \| null | Optional description. |
| `requires_function_calling` | boolean | When `true` (default), saving rejects rule/fallback targets whose model is known not to support function calling (`422 router_capability_unsupported`). |
| `sticky_window_seconds` | integer | Sticky-session window, default `300`, `≥ 0`. |
| `fallback` | object | Masked reference to the fallback config: `{ model_config_id, provider, model, key_hint, exists }`. `exists: false` flags a dangling reference. |
| `rules` | array | Rule objects: `{ id, name, type, target, conditions, invalid }`. `target` is a masked reference like `fallback`. A stored rule that no longer parses is surfaced with `invalid: true` rather than erroring. |
| `created_at` / `updated_at` | string | ISO-8601 timestamps. |

Referenced configs are always returned **masked** — never decrypted key material.

## `POST /v1/model-routers`

Create a router.

**Request body**

| Field | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `name` | string | Yes | — | 1–255 chars. |
| `description` | string \| null | No | `null` | |
| `fallback_model_config_id` | string | Yes | — | Must exist (`422 validation_error` otherwise). |
| `requires_function_calling` | boolean | No | `true` | |
| `sticky_window_seconds` | integer | No | `300` | `≥ 0`. |
| `rules` | array | No | `[]` | Rule objects (see schema above). Every rule target must exist. |

```bash
curl -H "Authorization: Bearer $TOKEN" -X POST \
  http://localhost:8000/v1/model-routers \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "Cost-aware router",
    "fallback_model_config_id": "mcfg_cheap01",
    "rules": [
      {
        "name": "Long conversations to the big model",
        "type": "calculated",
        "model_config_id": "mcfg_7f3a9c21",
        "conditions": {
          "operator": "AND",
          "clauses": [
            { "property": "conversation_token_count", "comparator": "gt", "value": "4000" }
          ]
        }
      }
    ]
  }'
```

**Response** `201` — the router object. Errors: `422 validation_error` (missing
target config, with `details.model_config_id`), `422 router_capability_unsupported`
(a target model is known not to support function calling while
`requires_function_calling` is on).

## `GET /v1/model-routers`

List routers, most-recent first. Paginated — see
[Pagination](/docs/api-reference/overview#pagination).

## `GET /v1/model-routers/{router_id}`

Fetch a single router. Unknown id returns `404`.

## `PATCH /v1/model-routers/{router_id}`

Partially update a router (PATCH semantics — only sent fields change). The
effective post-update rule set and fallback are re-validated. If `rules` is
omitted, existing rules carry forward; stored rules that no longer parse don't
block an unrelated edit (they stay inert and flagged `invalid`).

## `DELETE /v1/model-routers/{router_id}`

Delete a router. Agents referencing it get `router_id` **nulled** and fall back
to their plain model settings.

**Response** `204` — no body. Unknown id returns `404`.

## Assigning a router to an agent

Set `router_id` on agent create or PATCH (see
[Agents](/docs/api-reference/agents)). Rules:

- `router_id` and `model_config_id` are mutually exclusive
  (`422 model_source_conflict`); setting one via PATCH clears the other.
- Assigning a router to an agent with tools enabled (or changing tools while a
  router is assigned) is blocked if any reachable target model is known not to
  support function calling (`422 router_capability_unsupported`).
- At turn time, a dangling `router_id` degrades gracefully to the agent's plain
  model settings rather than failing the turn.
