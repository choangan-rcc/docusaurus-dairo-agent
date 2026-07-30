---
sidebar_position: 10
title: Guardrails
---

# Guardrails

Guardrails check what goes into and comes out of an agent, blocking or
rewriting content that violates policy. The platform integrates
[NVIDIA NeMo Guardrails](https://github.com/NVIDIA/NeMo-Guardrails) as its
checking engine, running in-process on every guarded turn.

Guardrails have two switches, and **both** must be on for checks to run:

1. **Platform-level** — an operator enables the NeMo integration via
   environment variables (`NEMO_GUARDRAIL_ENABLED=true` plus the model that
   powers the checks).
2. **Per-agent** — each agent has its own guardrails on/off switch and a
   **fallback message** returned when a check blocks a turn.

## What gets checked

When enabled, the pipeline can run these rails (each configured by the
operator):

- **Self-check input & output** — always on when guardrails are enabled: an LLM
  judges whether the user message / agent reply is appropriate.
- **Content safety** (input and output) — a dedicated safety model (e.g.
  NVIDIA's NemoGuard content-safety model) classifying against a 23-category
  harm taxonomy.
- **Topic control** (input) — keeps conversations within allowed subject areas.
- **Jailbreak detection** (input) — via NVIDIA's NemoGuard JailbreakDetect NIM
  or built-in perplexity heuristics.

## What happens on a block

- A blocked **input**: the model is never called. The agent replies with the
  fallback message, and both the user's message and the fallback reply are kept
  in history for auditability. The turn is marked with a `guardrail` stop
  reason.
- A blocked **output**: the model's reply is replaced with the fallback message.
- A **transform** (e.g. masking): the transformed text is what the model sees /
  the user receives, while the original stays stored.

Fallback precedence: the agent's own fallback message, then the checker's, then
a platform default.

## Failure behavior

By default guardrails **fail open** (`NEMO_GUARDRAIL_FAIL_OPEN=true`): if the
checking backend is unreachable, the turn proceeds and the miss is recorded.
Set it to `false` to fail closed — blocking every turn the checker can't
evaluate.

## Platform configuration

| Env var | Default | Purpose |
| --- | --- | --- |
| `NEMO_GUARDRAIL_ENABLED` | `false` | Master switch for the NeMo integration. |
| `NEMO_GUARDRAIL_MODEL` / `NEMO_GUARDRAIL_MODEL_ENGINE` | `gpt-4o-mini` / `openai` | The LLM powering self-check rails (plus `_BASE_URL`, `NEMO_GUARDRAIL_API_KEY`). |
| `NEMO_GUARDRAIL_CONTENT_SAFETY_MODEL` / `_ENGINE` / `_BASE_URL` | — | Enable the content-safety rail (e.g. `nvidia/llama-3.1-nemoguard-8b-content-safety`). |
| `NEMO_GUARDRAIL_TOPIC_CONTROL_MODEL` / `_ENGINE` / `_BASE_URL` | — | Enable the topic-control rail. |
| `NEMO_GUARDRAIL_JAILBREAK_NIM_URL` | — | Jailbreak detection via a NemoGuard NIM (key from `NVIDIA_API_KEY`). |
| `NEMO_GUARDRAIL_JAILBREAK_HEURISTICS` | `false` | Jailbreak detection via perplexity heuristics instead. |
| `NEMO_GUARDRAIL_FAIL_OPEN` | `true` | Fail-open vs fail-closed when the checker errors. |

NeMo Guardrails is an optional dependency — install with `pip install .[nemo]`
(or the project's extras equivalent).

## Per-agent fields

On the agent form, guardrails offer the on/off switch, the fallback message,
and a list of **forbidden topics** (passed to the checker as policy context).
Two further fields — a max-turns cap and an escalation webhook — are stored on
the agent today but **not yet enforced at runtime**; they are placeholders for
upcoming behavior.
