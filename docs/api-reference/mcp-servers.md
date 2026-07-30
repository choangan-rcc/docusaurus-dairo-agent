---
sidebar_position: 11
title: MCP Servers
---

# MCP Servers

An **MCP server** is a registered [Model Context Protocol](https://modelcontextprotocol.io)
endpoint whose tools an agent can call. Registration is a two-step flow: first
create the server record (it starts `pending`), then run **discovery**, which
connects to the server, runs the MCP `tools/list` handshake, and caches the tool
catalog. Servers are registered per workspace; **attaching one is agent-scoped**
— each agent picks a specific, non-empty subset of a server's discovered tools.

**How attached servers are used at runtime.** An agent's effective tool list for
a turn is the union of its built-in tools plus the tools selected on each
attached MCP server; only servers with `status: "active"` contribute. The LLM
sees each MCP tool's cached schema from `discovered_tools` — listing tools never
triggers a network call. A short-lived connection to the MCP server is opened
only when the LLM actually invokes one of its tools. If that connection fails,
that single tool call fails gracefully (the agent's turn continues) and the
server is flipped to `status: "error"` — it does not abort the chat turn.

## Config shapes

The `config` object's shape depends on `server_type`. A missing required field
is a `422 validation_error`.

| `server_type` | Field | Type | Required | Default | Constraints | Description |
| --- | --- | --- | --- | --- | --- | --- |
| `sse` / `streamable_http` | `url` | string | Yes | — | ≥ 1 char | Server endpoint URL. |
| | `headers` | object | No | `{}` | | Extra HTTP headers (e.g. auth). |
| | `timeout` | integer | No | `30` | `1 ≤ x ≤ 300` | Connection timeout, seconds. |
| `stdio` | `command` | string | Yes | — | ≥ 1 char | Executable to launch (e.g. `npx`). |
| | `args` | array\<string\> | No | `[]` | | Command arguments. |
| | `env` | object | No | `{}` | | Environment variables for the process. |

## MCP server object (`MCPServerOut`)

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | MCP server id (prefix `mcp_`). |
| `name` | string | The label you provided. |
| `description` | string \| null | Optional description. |
| `server_type` | string | `sse`, `streamable_http`, or `stdio`. |
| `config` | object | The stored transport config. |
| `discovered_tools` | array | Cached tool catalog: `{ name, description, input_schema }`. Empty until the first successful discovery. |
| `approval_tool_names` | array\<string\> | Tools on this server that pause a turn for human approval. See [Approval gating](#approval-gating). |
| `status` | string | `pending` (not yet discovered), `active` (discovery succeeded), or `error` (last discovery/connection failed). |
| `created_at` / `updated_at` | string | ISO-8601 timestamps. |

## `POST /v1/mcp-servers`

Register an MCP server. The record starts `pending` with empty
`discovered_tools` — no connection is made yet. Run discovery next.

```bash
curl -H "Authorization: Bearer $TOKEN" -X POST \
  http://localhost:8000/v1/mcp-servers \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "Weather MCP",
    "description": "Public weather tools",
    "server_type": "streamable_http",
    "config": {
      "url": "https://mcp.example.com/mcp",
      "headers": { "Authorization": "Bearer xyz" },
      "timeout": 30
    }
  }'
```

**Response** `201` — the server object with `status: "pending"`.

## `GET /v1/mcp-servers`

List registered servers. Paginated — see
[Pagination](/docs/api-reference/overview#pagination).

## `GET /v1/mcp-servers/{mcp_server_id}`

Fetch a single server. Unknown id returns `404`.

## `PATCH /v1/mcp-servers/{mcp_server_id}`

Partially update a server (`name`, `description`, `server_type`, `config`,
`approval_tool_names` — all optional). **Changing `config` invalidates the
previous discovery**: `status`
resets to `pending` and `discovered_tools` clears to `[]`, so run discovery
again before the server's tools can be used.

## `DELETE /v1/mcp-servers/{mcp_server_id}`

Delete a server registration. **Response** `204`.

## `POST /v1/mcp-servers/{mcp_server_id}/discover`

Connect to the server, run the MCP `tools/list` handshake, and refresh
`discovered_tools`. **This is the only endpoint that talks to the MCP server
over the network** — every other route is DB bookkeeping. On success, `status`
becomes `active`. On any connection/handshake failure, `status` becomes
`error`, `discovered_tools` clears, and the response is `502` with code
`mcp_discovery_failed`.

```bash
curl -H "Authorization: Bearer $TOKEN" -X POST \
  http://localhost:8000/v1/mcp-servers/mcp_3f8a1b20/discover
```

**Response** `200` — the refreshed server object with populated
`discovered_tools`.

## Approval gating

`approval_tool_names` on the **server** (settable on create and update) marks
tools that pause a turn for human approval before they run — the gate lives with
the connection rather than on each agent, so registering a server once carries
its risk profile to every agent that attaches it.

`agents.approval_tool_names` covers built-in tools only and **rejects** MCP tool
names; put them here instead. At run time a gated tool emits an
`approval-required` event — see
[Human-in-the-loop approvals](/docs/api-reference/chat#human-in-the-loop-approvals).

## Attaching servers to agents

### `GET /v1/agents/{agent_id}/mcp-servers`

List the attachments for an agent. Returns a **plain array** of attachment
objects:

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | Attachment id (prefix `ams_`). |
| `mcp_server_id` | string | The attached server. |
| `tool_names` | array\<string\> | The tools selected for this agent. |
| `created_at` | string | ISO-8601 timestamp. |

### `POST /v1/agents/{agent_id}/mcp-servers`

Attach a server with an **explicit, non-empty list of tools** — there is no
"attach the whole server" option. Every name must already exist in the server's
`discovered_tools`, so run discovery first.

**Idempotent update:** calling this again for an agent + server pair that is
already attached updates the tool selection in place (same attachment `id`).

**Request body**

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `mcp_server_id` | string | Yes | |
| `tool_names` | array\<string\> | Yes | ≥ 1 item; each must exist in `discovered_tools`. |

Unknown tool names return `422` with code `mcp_unknown_tools` listing the
offending names.

### `DELETE /v1/agents/{agent_id}/mcp-servers/{mcp_server_id}`

Detach a server from an agent. `204` on success; `404` if no such attachment
exists.
