---
sidebar_position: 7
title: Patterns
---

# Patterns

A **pattern** is a named, versioned recipe for assembling a runnable agent. The
catalog is synced from code at startup; admins can enable/disable patterns but
cannot create or delete them. Two patterns ship built in:

| Pattern id | Display name | When to use |
| --- | --- | --- |
| `function_agent` | Function Calling Agent | Default. Uses the model's native tool-calling API. Best for models that support function calling (Claude, GPT, Gemini). Requires a function-calling model at build time. |
| `react_agent` | ReAct Agent | Reason-then-act loop driven by text prompting. Works with any chat model; more prompt overhead, less reliable argument parsing. |

Each pattern exposes a `config_schema` — the JSON Schema of its config model,
used to validate an agent's `pattern_config`. All patterns share the same safety
fields:

| Field | Type | Required | Default | Constraints | Description |
| --- | --- | --- | --- | --- | --- |
| `max_iterations` | integer | No | `10` | `1 ≤ x ≤ 50` | Maximum agent reasoning/tool-call iterations per turn. |
| `timeout_seconds` | number | No | `120` | `0 < x ≤ 600` | Wall-clock ceiling for a single turn, in seconds. |

Unknown fields in `pattern_config` are ignored (dropped) rather than rejected, so
agents keep working across pattern upgrades.

## `GET /v1/patterns`

List catalog patterns. By default only enabled patterns are returned. Returns a
**plain array** (not paginated).

**Query parameters**

| Name | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `include_disabled` | boolean | No | `false` | If `true`, include disabled patterns as well. |

```bash
curl -H "Authorization: Bearer $TOKEN" http://localhost:8000/v1/patterns
```

**Response** `200`:

```json
[
  {
    "id": "function_agent",
    "display_name": "Function Calling Agent",
    "description": "Uses the model's native function/tool-calling API to pick and invoke tools. Best default for models that support function calling (Claude, GPT, Gemini): most reliable tool selection, structured arguments, lowest prompt overhead.",
    "system_type": "agent",
    "config_schema": {
      "type": "object",
      "title": "FunctionAgentConfig",
      "additionalProperties": false,
      "properties": {
        "max_iterations": {
          "type": "integer",
          "default": 10,
          "minimum": 1,
          "maximum": 50
        },
        "timeout_seconds": {
          "type": "number",
          "default": 120.0,
          "exclusiveMinimum": 0,
          "maximum": 600
        }
      }
    },
    "is_enabled": true,
    "version": 1,
    "updated_at": "2026-07-14T09:00:00Z"
  }
]
```

**Pattern object fields**

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | Pattern id (matches the `pattern_id` used on agents). |
| `display_name` | string | Human-friendly name. |
| `description` | string | Description with "when to use" guidance. |
| `system_type` | string | `"agent"` or `"workflow"`. Built-ins are `"agent"`. |
| `config_schema` | object | JSON Schema for the pattern's `pattern_config`. |
| `is_enabled` | boolean | Whether the pattern can be assigned to new agents. |
| `version` | integer | Pattern code version (snapshotted onto agents at creation). |
| `updated_at` | string | ISO-8601 timestamp of the last catalog sync/toggle. |

## `PATCH /v1/patterns/{pattern_id}`

Enable or disable a pattern. There is no delete — patterns are only soft-toggled
so that agents referencing them keep a valid reference.

**Request body**

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `is_enabled` | boolean | Yes | New enabled state for the pattern. |

```bash
curl -H "Authorization: Bearer $TOKEN" -X PATCH \
  http://localhost:8000/v1/patterns/react_agent \
  -H 'Content-Type: application/json' \
  -d '{"is_enabled": false}'
```

**Response** `200` — the updated pattern object. Unknown `pattern_id` returns
`404` with code `not_found`.
