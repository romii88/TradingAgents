## ADDED Requirements

### Requirement: Moonshot provider can be selected as a first-class LLM provider
The system SHALL support `moonshot` as a first-class LLM provider for TradingAgents configuration and graph initialization.

#### Scenario: Graph is configured with Moonshot provider
- **WHEN** a TradingAgents configuration sets `llm_provider` to `moonshot`
- **THEN** the LLM client factory MUST return an OpenAI-compatible client configured for Moonshot

### Requirement: CLI exposes Moonshot provider selection
The CLI SHALL allow users to select Moonshot Kimi directly from the provider selection flow.

#### Scenario: User selects provider in the CLI
- **WHEN** the user opens the provider selection step
- **THEN** the CLI MUST display `Moonshot / Kimi` as a selectable provider option

### Requirement: Moonshot provider uses official API defaults
The Moonshot provider SHALL use the official Moonshot API endpoint and authentication environment variable by default.

#### Scenario: Moonshot client is created without a custom endpoint
- **WHEN** the system creates a client for the `moonshot` provider
- **THEN** it MUST use `https://api.moonshot.cn/v1` as the default base URL
- **AND** it MUST read the API key from `MOONSHOT_API_KEY`

### Requirement: Moonshot model selection supports a recommended first-release model
The system SHALL offer a recommended Moonshot model for both quick and deep CLI model selection in the first release.

#### Scenario: User selects quick or deep Moonshot model
- **WHEN** the selected provider is `moonshot`
- **THEN** the quick model list MUST include `kimi-k2.5`
- **AND** the deep model list MUST include `kimi-k2.5`
