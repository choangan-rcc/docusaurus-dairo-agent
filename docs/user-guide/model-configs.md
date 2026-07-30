---
sidebar_position: 8
title: Model Configs (BYOK)
---

# Model Configs (BYOK)

A model config is a **bring-your-own-key** credential: a provider, a model, and
your API key, stored encrypted and pinnable to any agent. Without a model
config, agents fall back to the server's environment-configured provider (the
offline `fake` provider by default).

## Supported providers

Native integrations for **Anthropic**, **Amazon Bedrock**, and **Google
Gemini**, plus **OpenAI** and a family of OpenAI-compatible services — Groq,
Together AI, DeepSeek, OpenRouter, xAI (Grok), Fireworks AI, local **Ollama**
(no key needed), and a custom **OpenAI-compatible** option for anything else
(vLLM, LM Studio, LiteLLM proxies, …).

Provider quirks the form handles for you:

- **Bedrock** takes AWS credentials as `ACCESS_KEY_ID:SECRET_ACCESS_KEY` and an
  AWS region instead of a base URL.
- OpenAI-compatible services get their default base URL pre-filled; only the
  custom option requires you to enter one.

## Adding a key

From the **Model Configs** page, pick a provider from the catalog (or hit
"Add"), then:

1. Name the config and enter your API key.
2. Click **Load models** — the platform lists the models your key can actually
   access (live from the provider, with a static fallback when the provider is
   unreachable), including context-window and tool-support hints.
3. Pick a model and save.

Saving **verifies the key with a real provider call first** — an invalid key is
rejected and nothing is stored. Stored keys are encrypted with AES-256-GCM and
never shown again; everywhere in the UI and API you only see a hint like
`…def4`. Each config also tracks when it was last used by a live conversation.

## Using a config

Pin a config to an agent in the agent form's model picker. An agent uses either
a pinned config **or** a [model router](/docs/user-guide/model-routers) — never
both.

## Deleting a key

Deleting a config never breaks a running agent: agents that referenced it
automatically fall back to the server's default provider. If any model routers
still reference the config, the UI warns you — those routers show a "needs
repair" badge until you point them at another config.

## API

Full request/response details are in
[Model Configs](/docs/api-reference/model-configs) and
[Providers](/docs/api-reference/providers).
