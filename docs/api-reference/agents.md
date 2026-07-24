---
sidebar_position: 4
title: Agents
---

# Agents

An **agent** bundles a system prompt, model settings (a pinned BYOK config *or*
a model router), a pattern, tools (built-in, MCP, and knowledge-base retrieval),
approval gates, and guardrails. All agent routes require auth and are
workspace-scoped.

## Agent object (`AgentOut`)

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | Agent id (prefix `agt_`). |
| `name` | string | Agent name. |
| `avatar` | string \| null | Optional avatar URL/reference. |
| `description` | string \| null | Optional description. |
| `system_prompt` | string | System prompt (may contain `{{variables}}`). |
| `template_variables` | object | Values for `{{variables}}` in the prompt. |
| `model` | string | Model id (defaults to `claude-sonnet-4-6`). |
| `temperature` | number | Sampling temperature (`0.0`–`2.0`). |
| `max_tokens` | integer | Max output tokens (`≥ 1`). |
| `language` | string | Language code (e.g. `en`). |
| `status` | string | Lifecycle status (`draft`, `active`, …). New agents start as `draft`. |
| `model_config_id` | string \| null | Pinned BYOK model config, or `null`. Mutually exclusive with `router_id`. |
| `router_id` | string \| null | Assigned [model router](/docs/api-reference/model-routers), or `null`. |
| `pattern_id` | string | Agent pattern id. |
| `pattern_config` | object | Normalized pattern config (validated against the pattern schema). |
| `pattern_version` | integer | Pattern code version snapshotted at write time. |
| `tool_names` | array\<string\> | Enabled built-in tool names. |
| `approval_tool_names` | array\<string\> | Tools requiring human approval before each call (must be a subset of `tool_names`). |
| `guardrails` | object | Guardrails config (see below). |
| `created_at` / `updated_at` | string | ISO-8601 timestamps. |

**Guardrails object** (`guardrails`)

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `enabled` | boolean | `false` | Per-agent opt-in. Guardrails only run when this **and** the platform flag `NEMO_GUARDRAIL_ENABLED` are both on. |
| `forbidden_topics` | array\<string\> | `[]` | Topics the agent should refuse. Stored and passed to the checker as policy context; not directly enforced yet. |
| `fallback_message` | string | *(a default apology/handoff message)* | Message returned when a check blocks the turn. |
| `max_turns` | integer | `20` | Max conversation turns (stored; not yet enforced at runtime). |
| `escalation` | object | `{ "enabled": false, "webhook_url": null }` | Escalation webhook config (URL is SSRF-validated at write time; firing is not yet implemented). |

Built-in tools available for `tool_names` are served by
[`GET /v1/tools`](/docs/api-reference/tools) — `calculator`, `current_time`, and
a set of Yahoo Finance tools. Unknown tool names are rejected with `422` at
write time.

## `GET /v1/agents/templates`

List the seeded agent templates for the create-agent picker. Returns a plain
array. Available template keys: `customer_support`, `internal_knowledge`,
`sales_assistant`, `document_analyst`.

```bash
curl -H "Authorization: Bearer $TOKEN" http://localhost:8000/v1/agents/templates
```

**Template object fields**

| Field | Type | Description |
| --- | --- | --- |
| `key` | string | Template key (pass as `?template=<key>` on create). |
| `name` / `description` | string | What the template is for. |
| `model` / `temperature` / `max_tokens` / `language` | — | Suggested model settings. |
| `system_prompt` | string | Pre-written prompt (may contain `{{vars}}`). |
| `guardrails` | object | Default guardrails. |

## `GET /v1/agents`

List agents, most-recent first. Paginated.

**Query parameters**

| Name | Type | Default | Description |
| --- | --- | --- | --- |
| `status` | string \| null | — | Filter by lifecycle status (e.g. `draft`). |
| `limit` / `offset` | integer | `20` / `0` | See [Pagination](/docs/api-reference/overview#pagination). |

## `POST /v1/agents`

Create an agent. Two modes:

- **From scratch** (no `?template`): a body is required and `name` is mandatory;
  everything else falls back to defaults.
- **From a template** (`?template=<key>`): pre-filled from the template; body
  fields override template defaults. The body is optional.

**Request body** (all optional except `name` when not using a template)

| Field | Type | Default | Constraints / Notes |
| --- | --- | --- | --- |
| `name` | string | — | 1–255 chars. Required without a template. |
| `description` / `avatar` | string \| null | `null` | |
| `system_prompt` | string | `""` | |
| `template_variables` | object | `{}` | Values for `{{vars}}` in the prompt. |
| `model` | string | `claude-sonnet-4-6` | |
| `temperature` | number | `0.7` | `0.0`–`2.0` |
| `max_tokens` | integer | `1024` | `≥ 1` |
| `language` | string | `en` | |
| `model_config_id` | string \| null | `null` | Must exist (`422`). Mutually exclusive with `router_id` (`422 model_source_conflict`). |
| `router_id` | string \| null | `null` | Must exist (`422`). |
| `pattern_id` | string | `function_agent` | Must exist and be enabled (`422`). |
| `pattern_config` | object | `{}` | Validated against the pattern schema; stored normalized. |
| `tool_names` | array\<string\> | `["calculator", "current_time"]` | Unknown names rejected (`422`). |
| `approval_tool_names` | array\<string\> | `[]` | Must be a subset of `tool_names` (`422`). |
| `guardrails` | object | *(defaults)* | |

`status` cannot be set on create — new agents always start as `draft`.

```bash
curl -H "Authorization: Bearer $TOKEN" -X POST \
  'http://localhost:8000/v1/agents?template=customer_support' \
  -H 'Content-Type: application/json' \
  -d '{ "name": "Acme Support", "temperature": 0.3 }'
```

**Response** `201` — the agent object, with `pattern_config` echoed back
normalized. Creating an agent also records **version 1** of its prompt history
(see [Versions](#agent-versions)).

## `GET /v1/agents/{agent_id}`

Fetch a single agent. Unknown id returns `404`.

## `PATCH /v1/agents/{agent_id}`

Partially update an agent — only sent fields change. Accepts the same fields as
create, all optional, plus `status`.

Notes:

- Changing `pattern_id` or `pattern_config` re-validates the effective config
  and re-snapshots `pattern_version`.
- Setting `model_config_id` clears `router_id` and vice versa.
- Assigning a router while the agent has tools (or changing tools with a router
  assigned) is blocked if any router target model is known not to support
  function calling (`422 router_capability_unsupported`).
- If `system_prompt` or `template_variables` actually changed, a new prompt
  version is recorded automatically.

```bash
curl -H "Authorization: Bearer $TOKEN" -X PATCH \
  http://localhost:8000/v1/agents/agt_a1b2c3d4 \
  -H 'Content-Type: application/json' \
  -d '{ "status": "active", "temperature": 0.2 }'
```

## `DELETE /v1/agents/{agent_id}`

Delete an agent. Its usage events are kept but folded into the deleted-agents
bucket. **Response** `204`.

## `POST /v1/agents/{agent_id}/duplicate`

Deep-copy an agent's configuration into a new `draft` named
`Copy of <original name>`. **Response** `201` — the new agent (fresh `id`,
`status: "draft"`, its own version 1).

## Agent versions

Prompt changes are versioned automatically — an append-only snapshot history of
`system_prompt` + `template_variables`. There is no publish step; versions are
recorded on create, duplicate, and any PATCH that actually changes the prompt or
its variables.

### `GET /v1/agents/{agent_id}/versions`

Paginated version list, newest first.

**Version object fields**

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | Version id (prefix `ver_`). |
| `agent_id` | string | Owning agent. |
| `version_number` | integer | Monotonic per agent. |
| `config_snapshot` | object | `{ "system_prompt": "...", "template_variables": {...} }` |
| `changelog` | string \| null | E.g. `"Initial version"`, `"System prompt updated"`, `"Restored from version 3"`. |
| `created_at` | string | ISO-8601 timestamp. |

### `POST /v1/agents/{agent_id}/versions/{version_number}/restore`

Copy a snapshot's prompt + variables back onto the agent. History is never
rewritten — restoring records a **new** version with changelog
`"Restored from version <n>"`. Restoring the current state is a no-op.

**Response** `200` — the updated agent object. Unknown agent or version returns
`404`.
