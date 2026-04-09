## Context

TradingAgents separates provider integration from agent orchestration. `TradingAgentsGraph` only depends on the provider name, model name, and a LangChain-compatible chat model returned by the LLM client layer. This makes Kimi support a provider-layer extension rather than a graph or prompt redesign.

The current code already uses `OpenAIClient` for multiple OpenAI-compatible providers such as `xai`, `openrouter`, and `ollama`. That existing pattern is the lowest-risk way to add Moonshot support quickly while preserving CLI ergonomics and minimizing code churn.

## Goals / Non-Goals

**Goals:**
- Add `moonshot` as a formal provider that can be selected directly from the CLI.
- Use the official Moonshot API endpoint and `MOONSHOT_API_KEY`.
- Provide one recommended Kimi model, `kimi-k2.5`, for both quick and deep modes so users can run the graph immediately.
- Keep agent, graph, and dataflow logic unchanged.
- Add minimal documentation and test coverage so the provider is maintainable.

**Non-Goals:**
- Add custom endpoint override support for Moonshot in the first release.
- Add multiple Kimi model variants or provider-specific advanced tuning.
- Add Zhipu, DeepSeek, or other new providers in the same change.
- Redesign prompts, tool schemas, or the LangGraph orchestration flow.

## Decisions

### Reuse `OpenAIClient` for `moonshot`

`moonshot` will be routed through the existing `OpenAIClient` rather than a new `moonshot_client.py`. This matches the current architecture and keeps the implementation small. A dedicated client can be introduced later only if Moonshot-specific behavior requires it.

Alternative considered:
- Add a dedicated `moonshot_client.py`
  - Rejected for now because it would mostly duplicate existing OpenAI-compatible logic without a clear first-release benefit.

### Treat Moonshot as a first-class provider in the CLI

The CLI will show `Moonshot / Kimi` alongside the existing providers. This is preferred over asking users to pretend Kimi is `openai` with a custom base URL because provider semantics stay clear and future docs/tests remain straightforward.

Alternative considered:
- Reuse `openai` provider with manual endpoint changes
  - Rejected because it creates confusing UX and weakens maintainability.

### Use the official API only in the first release

The first release will hard-wire Moonshot to `https://api.moonshot.cn/v1` and `MOONSHOT_API_KEY`. This keeps the behavior easy to explain and verify.

Alternative considered:
- Allow custom `backend_url`
  - Deferred because the user explicitly wants official API support first and faster delivery is more important than configurability in this iteration.

### Ship one recommended model first

Both CLI model groups, `quick` and `deep`, will list `kimi-k2.5` only. This optimizes for “get a full graph run working” over early model taxonomy.

Alternative considered:
- Offer separate fast/deep Moonshot models from day one
  - Deferred because model differentiation is less important than stable end-to-end support in the first release.

## Risks / Trade-offs

- Tool-calling compatibility in analyst nodes may differ from other OpenAI-compatible providers
  - Mitigation: verify factory/client wiring with tests first, then run an end-to-end smoke test when credentials are available.
- A single model for both quick and deep paths reduces flexibility
  - Mitigation: keep the model catalog structure unchanged so additional Moonshot models can be added later without redesign.
- Moonshot model names may evolve over time
  - Mitigation: keep the initial catalog minimal and update model entries independently from provider wiring if needed.
- The first release does not support custom endpoints or proxies
  - Mitigation: keep that explicitly out of scope so the implementation stays predictable.
