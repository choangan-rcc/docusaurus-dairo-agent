---
sidebar_position: 6
title: Built-in Tools
---

# Built-in Tools

Built-in tools run in-process and are enabled per agent via `tool_names`. The
catalog is served from the code registry so clients never hardcode it.

## `GET /v1/tools`

List the built-in tool catalog. Returns a plain array of
`{ "name": string, "description": string }`.

```bash
curl -H "Authorization: Bearer $TOKEN" http://localhost:8000/v1/tools
```

## Available tools

| Name | Description |
| --- | --- |
| `calculator` | Evaluate a basic arithmetic expression (`+ - * / ** %`, unary minus, numeric literals only — a safe AST evaluator, no `eval`). |
| `current_time` | Return the current UTC time as an ISO-8601 string. |
| `yahoo_finance_balance_sheet` | Balance sheet for a ticker. |
| `yahoo_finance_income_statement` | Income statement for a ticker. |
| `yahoo_finance_cash_flow` | Cash flow statement for a ticker. |
| `yahoo_finance_stock_basic_info` | Basic stock info (price, market cap, …). |
| `yahoo_finance_stock_analyst_recommendations` | Analyst recommendations. |
| `yahoo_finance_stock_news` | Recent news for a ticker. |

New agents default to `["calculator", "current_time"]`.

Notes:

- Tool failures return an error **string** to the model (never raise), so a
  failing tool degrades the answer, not the turn.
- An agent configured with a tool name that has since been removed from the
  registry still answers — unknown names are skipped at runtime (they are
  rejected at *write* time, though).
- Any built-in tool can be gated behind human approval per agent via
  `approval_tool_names` — see
  [Human-in-the-loop approvals](/docs/api-reference/chat#human-in-the-loop-approvals).
- Knowledge-base retrieval and MCP tools are separate mechanisms and not part of
  this catalog — see [Knowledge Bases](/docs/api-reference/knowledge-bases) and
  [MCP Servers](/docs/api-reference/mcp-servers).
