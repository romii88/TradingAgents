## Why

TradingAgents already has a clean provider abstraction, but Kimi cannot yet be selected as a first-class provider from the CLI. We want the first incremental China-market-friendly extension to be small, reliable, and able to run the full graph quickly without changing agent orchestration.

## What Changes

- Add `moonshot` as a formal LLM provider in the provider factory.
- Expose `Moonshot / Kimi` directly in the CLI provider selector.
- Route the provider to the official Moonshot API endpoint and `MOONSHOT_API_KEY`.
- Offer a single recommended Kimi model, `kimi-k2.5`, for both quick and deep selection modes in the first release.
- Add documentation and test coverage for the new provider path.

## Capabilities

### New Capabilities
- `moonshot-provider-support`: Support selecting and running TradingAgents with Moonshot Kimi as a first-class provider.

### Modified Capabilities

## Impact

- Affected code:
  - `tradingagents/llm_clients/openai_client.py`
  - `tradingagents/llm_clients/factory.py`
  - `tradingagents/llm_clients/model_catalog.py`
  - `cli/utils.py`
  - `README.md`
  - `tests/test_model_validation.py`
- External API:
  - Moonshot official API at `https://api.moonshot.cn/v1`
- Environment variables:
  - `MOONSHOT_API_KEY`
