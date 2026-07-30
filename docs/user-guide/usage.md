---
sidebar_position: 13
title: Usage & Costs
---

# Usage & Costs

Every assistant turn records a usage event — tokens in and out, wall-clock
latency, and an estimated cost derived from a per-model pricing table. The
**Usage** page aggregates these into a workspace dashboard; no setup is
required.

## The dashboard

Pick a trailing window of **7, 30, or 90 days**. The page shows:

- **Stat tiles** — total requests, total tokens (with the in/out split),
  estimated cost, and average latency for the window.
- **Cost over time** — a per-day cost chart across the window.
- **By agent** — requests, tokens, cost, and average latency per agent, sorted
  by cost. Agents link to their configuration page; usage from since-deleted
  agents is kept and grouped under "(deleted agents)".
- **By model** — the same breakdown per model, which is where
  [model-router](/docs/user-guide/model-routers) savings become visible.

A search box filters both breakdown tables, and each table shows the top 50
rows by cost.

## Notes on the numbers

- Costs are **estimates** from a static pricing table (USD per million tokens);
  models missing from the table count as $0.
- Days are bucketed in UTC.
- The dashboard is read-only — quotas and billing are out of scope for now.

For programmatic access, see the
[usage API](/docs/api-reference/observability-and-usage#usage).
