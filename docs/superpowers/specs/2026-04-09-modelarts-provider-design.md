# Huawei Cloud / DeepSeek Provider Design

## Goal

Add a first-class `modelarts` LLM provider to TradingAgents so the CLI can directly select a fixed Huawei Cloud / DeepSeek deployment without requiring custom URL or custom model input.

This first release is intentionally narrow:

- Provider id: `modelarts`
- CLI label: `Huawei Cloud / DeepSeek`
- Base URL: `https://api.modelarts-maas.com/openai/v1`
- Model: `deepseek-v3.1-terminus`
- API key env var: `MODELARTS_API_KEY`

## Why This Shape

The current codebase already supports several OpenAI-compatible providers through `OpenAIClient`, including OpenRouter and Moonshot. Huawei Cloud ModelArts DeepSeek fits the same architectural seam:

- provider-specific base URL
- provider-specific API key env var
- standard chat completions interface

Because of that, the best first implementation is a new formal provider that reuses the existing `OpenAIClient` path instead of introducing a separate client or a generic custom gateway abstraction.

## Scope

### In Scope

- Add a formal `modelarts` provider routed through `OpenAIClient`
- Expose `Huawei Cloud / DeepSeek` in the CLI provider selector
- Add a fixed model entry for both quick and deep model slots:
  - `deepseek-v3.1-terminus`
- Read credentials from `MODELARTS_API_KEY`
- Document setup in README and `.env.example`
- Add tests for provider creation and model validation behavior

### Out of Scope

- User-specified base URLs
- User-specified model ids
- A generic `custom_openai_compatible` provider
- Writing any real API key to tracked files
- Reworking graph logic, agent prompts, or tool orchestration

## Implementation Plan

### 1. Provider registration

Update the OpenAI-compatible provider registry so `modelarts` maps to:

- base URL: `https://api.modelarts-maas.com/openai/v1`
- env var: `MODELARTS_API_KEY`

`factory.py` should route `modelarts` to `OpenAIClient`, the same way existing compatible providers are handled.

### 2. CLI exposure

Add `Huawei Cloud / DeepSeek` to the provider selector in the CLI.

Because this first version is fixed-shape, the CLI should not prompt for a custom URL or a custom model when `modelarts` is selected. Instead, the user should see the shared catalog entry and choose the fixed model normally.

### 3. Model catalog

Add one `modelarts` entry to the shared model catalog:

- quick: `deepseek-v3.1-terminus`
- deep: `deepseek-v3.1-terminus`

This keeps the CLI consistent with the rest of the providers and makes validation deterministic.

### 4. Documentation

Update user-facing docs to include:

- `MODELARTS_API_KEY`
- `modelarts` in supported providers
- a short Python config example using:
  - `llm_provider = "modelarts"`
  - `quick_think_llm = "deepseek-v3.1-terminus"`
  - `deep_think_llm = "deepseek-v3.1-terminus"`

### 5. Tests

Add or extend tests to verify:

- `create_llm_client("modelarts", "deepseek-v3.1-terminus")` returns `OpenAIClient`
- the client provider value is `modelarts`
- `modelarts` catalog entries pass validation
- unknown custom model warnings still behave correctly for strict providers

## Validation Strategy

Validation should happen in two layers:

### Automated

- run `python -m unittest tests.test_model_validation -v`

### Manual smoke

- set `MODELARTS_API_KEY`
- perform a minimal invoke through the provider
- optionally run a light TradingAgents flow with a single analyst

## Risks

### 1. Provider works but data vendor blocks execution

This was already observed during manual testing with `yfinance` rate limits. That is a data-source issue, not a provider registration issue.

### 2. OpenAI-compatible behavior differs across gateways

The provider is expected to work because Huawei Cloud ModelArts exposed a compatible chat endpoint in manual smoke tests. If compatibility drifts later, the likely fix is provider-specific request tweaks inside `OpenAIClient`, not graph changes.

## Decision Summary

The first release should ship as a narrow, explicit provider:

- yes: `modelarts`
- yes: fixed Huawei Cloud endpoint
- yes: fixed `deepseek-v3.1-terminus`
- no: generic gateway configuration
- no: custom CLI URL/model input

This keeps the implementation small, stable, and aligned with the current architecture.
