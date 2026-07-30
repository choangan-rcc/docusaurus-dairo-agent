---
sidebar_position: 12
title: Data Sources
---

# Data Sources

A **data source** is a connection to one of your own systems — a PostgreSQL
database today — whose tools an agent can call. It is the managed counterpart to
an [MCP server](/docs/api-reference/mcp-servers): instead of registering a
server you host yourself, you pick a provider from a curated catalog and supply
credentials, and the platform launches a **version-pinned third-party MCP
server** on your behalf with those credentials injected in memory at run time.

The platform ships no tool code for data sources — every provider is an existing
MCP server described by one catalog entry (command, pinned package, env mapping,
health check, write-capable tools). That is what keeps arbitrary code execution
off the table: you supply values, never a command line.

Three invariants shape this whole surface:

- **Credentials never come back.** Secrets arrive in `POST`/`PATCH` bodies, go
  straight into an AES-GCM bundle, and are returned only as a masked
  `credential_hint`. No response contains a credential.
- **The launch template is internal.** The catalog omits the command, args, and
  env mapping used to start the connector — clients can neither read nor
  influence it.
- **Provider text is never echoed.** Driver and MCP failures routinely quote the
  rendered DSN (`authentication failed for connection 'postgresql://u:pw@…'`),
  so only a fixed set of platform phrases is ever stored in `status_reason` or
  returned in an error.

A failed connection attempt is a **stored state, not a 4xx**: the row is created
or updated with `status: "failed"` and its encrypted bundle intact, so you fix a
field and call `/test` instead of retyping the password.

## Lifecycle

```text
POST /v1/data-sources          create + probe        → connected | failed
POST /v1/data-sources/{id}/test    re-probe stored credentials
POST /v1/data-sources/{id}/discover refresh the tool catalog
POST /v1/agents/{id}/data-sources   attach to an agent
```

Creating a source probes it immediately: the platform starts the connector, runs
the provider's health tool, and caches the tool list — all in one MCP session.

## Statuses

`status` is what the platform last observed; `enabled` is what you chose.

| `status` | Meaning |
| --- | --- |
| `pending` | Row created, no successful observation yet. |
| `connected` | The last probe reached the system and listed its tools. |
| `failed` | The last probe failed; `status_reason` carries a fixed phrase. |
| `disconnected` | You set `enabled: false` — the source is not probed and not used. |

Only sources that are **both `enabled` and `connected`** contribute tools to a
chat turn; anything else is silently skipped.

`status_reason` is one of a bounded set of phrases:

| Phrase | Cause |
| --- | --- |
| `The connection attempt timed out.` | Timeout reaching the system. |
| `The connection was refused or the host is unreachable.` | Connection/OS error. |
| `The connector runtime could not be started.` | The pinned runner (`uvx`) is missing or not executable. |
| `The connector configuration is invalid.` | Template/value rendering problem. |
| `The connection could not be established.` | Anything else. |

## `GET /v1/data-source-providers`

The connectable-provider catalog. Read-only, workspace-independent, and the
source of truth for the dynamic connection form. Returns a **plain array**.

```bash
curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:8000/v1/data-source-providers
```

**Descriptor fields**

| Field | Type | Description |
| --- | --- | --- |
| `provider` | string | Stable provider key (e.g. `postgres`). |
| `display_name` | string | Human label. |
| `category` | string | `database`, `file_storage`, `messaging`, or `api`. |
| `auth_kind` | string | `fields`, `connection_string`, or `oauth2`. |
| `description` | string | One-line summary. |
| `fields` | array | The inputs to render — see below. |

**Field spec**

| Field | Type | Description |
| --- | --- | --- |
| `key` | string | The key to send inside `values`. |
| `label` | string | Form label. |
| `kind` | string | `text`, `password`, `number`, `select`, or `boolean`. |
| `required` | boolean | Whether a value must be supplied on create. |
| `secret` | boolean | `true` = stored encrypted and never returned. Render as a password input. |
| `placeholder` | string \| null | Optional hint. |
| `options` | array\<string\> | Allowed values for `kind: "select"`. |
| `default` | any | Default value, when the provider defines one. |

**Built-in providers**

| Provider | Backing MCP server | Fields |
| --- | --- | --- |
| `postgres` | `postgres-mcp` (pinned `0.3.0`) | `host`, `port` (default `5432`), `database`, `username` *(secret)*, `password` *(secret)*, `access_mode` (`restricted` \| `unrestricted`, default `restricted`) |

For `postgres`, `access_mode: "unrestricted"` is what makes `execute_sql`
write-capable; a `restricted` connection reports no write tools at all.

## Data source object (`DataSourceOut`)

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | Data source id (prefix `ds_`). |
| `name` | string | Your label; unique per workspace. |
| `provider` | string | Catalog provider key. |
| `config` | object | The **non-secret** stored values (e.g. `host`, `port`, `database`, `access_mode`). |
| `approval_tool_names` | array\<string\> | Tools on this connection that pause a turn for [human approval](/docs/api-reference/chat#human-in-the-loop-approvals). |
| `write_tool_names` | array\<string\> | Tools the current `config` can mutate data with. **Derived on read**, never stored — it changes the moment an access mode changes. |
| `credential_hint` | string \| null | Masked hint of the stored secrets. Never the value. |
| `status` | string | See [Statuses](#statuses). |
| `status_reason` | string \| null | Fixed failure phrase when `status: "failed"`. |
| `enabled` | boolean | Your on/off intent. |
| `discovered_tools` | array | Cached catalog: `{ name, description }`. Cleared by a failed probe. |
| `last_checked_at` | string \| null | Last probe (ISO-8601). |
| `last_used_at` | string \| null | Last time a turn used this source. |
| `created_at` / `updated_at` | string | ISO-8601 timestamps. |

## `POST /v1/data-sources`

Create a connection: validate the submitted fields against the descriptor,
encrypt the secrets, probe the connection, and store the row.

**Request body**

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `name` | string | Yes | 1–255 chars. Duplicate names return `422 datasource_validation_failed`. |
| `provider` | string | Yes | A `provider` from the catalog. |
| `values` | object | No | **One flat map** of all field values, secret and not. The descriptor decides the split — clients never pre-separate them. |
| `approval_tool_names` | array\<string\> \| null | No | Explicit approval gate. Omit to let the platform default it (see [Approval gating](#approval-gating)). |

```bash
curl -H "Authorization: Bearer $TOKEN" -X POST \
  http://localhost:8000/v1/data-sources \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "Analytics replica",
    "provider": "postgres",
    "values": {
      "host": "db.internal",
      "port": 5432,
      "database": "analytics",
      "username": "readonly",
      "password": "s3cret",
      "access_mode": "restricted"
    }
  }'
```

**Response** `201` — the data source object. Note that a probe failure still
returns `201`, with `status: "failed"` and a `status_reason`.

## `GET /v1/data-sources`

List the workspace's data sources. Paginated — see
[Pagination](/docs/api-reference/overview#pagination).

## `GET /v1/data-sources/{ds_id}`

Fetch one source. An unknown id — or one belonging to another workspace — is
`404`, never `403`.

## `PATCH /v1/data-sources/{ds_id}`

Partial update, and the endpoint that re-probes.

| Field | Type | Description |
| --- | --- | --- |
| `name` | string \| null | Rename (uniqueness still enforced). |
| `values` | object \| null | Only the fields being changed. **Blank or omitted secrets keep the stored bundle** — editing the host does not wipe the password. |
| `enabled` | boolean \| null | `false` sets `status: "disconnected"` and stops probing; `true` on a disconnected source re-probes it. |
| `approval_tool_names` | array\<string\> \| null | Replace the gate. `[]` is a deliberate "gate nothing" and is respected. |

The source is re-probed whenever `values` changed the connection, or when
`enabled` flips from `false` back to `true`.

```bash
curl -H "Authorization: Bearer $TOKEN" -X PATCH \
  http://localhost:8000/v1/data-sources/ds_7c1e9a44 \
  -H 'Content-Type: application/json' \
  -d '{"values": {"access_mode": "unrestricted"}}'
```

## `DELETE /v1/data-sources/{ds_id}`

Delete the source; its agent attachments are removed first. **Response** `204`.

## `GET /v1/data-sources/{ds_id}/agents`

Reverse lookup — the agents this source is attached to. Returns a **plain
array** of `{ id, name, status }`.

## `POST /v1/data-sources/{ds_id}/test`

Re-run validation with the **stored** credentials and record the outcome. Always
`200` with the updated object: a rejected password is a `status: "failed"` row,
not an error response.

```bash
curl -H "Authorization: Bearer $TOKEN" -X POST \
  http://localhost:8000/v1/data-sources/ds_7c1e9a44/test
```

## `POST /v1/data-sources/{ds_id}/discover`

Refresh `discovered_tools`. **Identical to `/test`** on purpose: listing tools
and the health call share one MCP session, so a refresh is always also an
observation of the connection — reporting a stale `status` afterwards would be a
lie.

## Approval gating

Gating for data-source tools lives on the **data source**, not on the agent, so
the decision sits beside the access mode that makes a tool write-capable.
(`agents.approval_tool_names` covers built-in tools only and rejects MCP or
data-source tool names.)

- Any **discovered** tool may be gated — not only the write-capable ones. A read
  tool on a production connection is a legitimate thing to want a human on.
- Unknown names return `422 datasource_validation_failed`; run discovery first.
- **Omitting** `approval_tool_names` on create or update keeps the stored gate
  **plus anything this request just made write-capable**. Switching a Postgres
  connection to `unrestricted` therefore gates `execute_sql` automatically,
  while an explicit `[]` waives it.

At run time a gated tool pauses the turn with an `approval-required` SSE event —
see [Human-in-the-loop approvals](/docs/api-reference/chat#human-in-the-loop-approvals).

## Attaching data sources to agents

### `GET /v1/agents/{agent_id}/data-sources`

The agent's attachments. Returns a **plain array**:

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | Attachment id. |
| `data_source_id` | string | The attached source. |
| `tool_names` | array\<string\> \| null | The selected tools, or `null` for "all discovered tools". |
| `created_at` | string | ISO-8601 timestamp. |

### `POST /v1/agents/{agent_id}/data-sources`

Attach a source to an agent.

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `data_source_id` | string | Yes | The source to attach. |
| `tool_names` | array\<string\> \| null | No | Subset of `discovered_tools`. **Omitted or `null` means every discovered tool** — the opposite default from an MCP server, because a data source's catalog is derived from its own schema. |

Names not present in `discovered_tools` return `422
datasource_validation_failed`. Re-attaching an already-attached pair updates the
tool selection in place. **Response** `201`.

```bash
curl -H "Authorization: Bearer $TOKEN" -X POST \
  http://localhost:8000/v1/agents/agt_a1b2c3d4/data-sources \
  -H 'Content-Type: application/json' \
  -d '{"data_source_id": "ds_7c1e9a44", "tool_names": ["list_schemas", "execute_sql"]}'
```

### `DELETE /v1/agents/{agent_id}/data-sources/{ds_id}`

Detach. `204` on success; `404` if no such attachment exists.

## How attached sources are used at run time

For each turn, the agent's effective tool list includes the selected tools of
every attached source that is `enabled` **and** `connected`. The LLM sees the
cached schemas — listing tools makes no network call. Credentials are decrypted
into the rendered launch config only when the connector is started for a turn.

Because two Postgres connections on one agent both advertise `execute_sql`, the
agent-facing tool name is prefixed:

```text
{slug-of-connection-name}_{id-suffix}__{remote_tool_name}
```

capped at 64 characters (the strictest provider limit). For example
`analytics_replica_7c1e9a44__execute_sql`. A source whose runtime config cannot
be rendered is skipped for that turn rather than failing it.

## Error codes

| Code | HTTP | When |
| --- | --- | --- |
| `datasource_validation_failed` | 422 | Field values don't match the descriptor; duplicate name; unknown tool names. |
| `datasource_auth_failed` | 422 | The backing system rejected the credentials. |
| `datasource_unreachable` | 422 | Spawn/connect/timeout failure before credentials were judged. |
| `datasource_disabled` | 409 | Operation attempted on a disabled or disconnected source. |
| `datasource_provider_unknown` | 500 | A stored row names a provider this build no longer ships. |
| `credential_encryption_misconfigured` | 500 | `CREDENTIAL_ENCRYPTION_KEY` missing or malformed. |
