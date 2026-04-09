# Support Kimi Provider Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add Moonshot Kimi as a first-class CLI-selectable provider using the official Moonshot API and one recommended first-release model.

**Architecture:** Reuse the existing OpenAI-compatible client path so the change stays inside the provider integration layer, CLI model selection, docs, and tests. Keep the LangGraph workflow and agent prompts unchanged, then validate the new provider through model validation tests and factory/client smoke coverage.

**Tech Stack:** Python, LangChain, Typer, Questionary, OpenSpec, unittest

---

### Task 1: Register Moonshot Provider Wiring

**Files:**
- Modify: `tradingagents/llm_clients/openai_client.py`
- Modify: `tradingagents/llm_clients/factory.py`
- Test: `tests/test_model_validation.py`

- [ ] **Step 1: Write the failing tests for provider creation and validation**

```python
def test_cli_catalog_models_are_all_validator_approved(self):
    for provider, models in get_known_models().items():
        if provider in ("ollama", "openrouter"):
            continue
        for model in models:
            with self.subTest(provider=provider, model=model):
                self.assertTrue(validate_model(provider, model))


def test_create_llm_client_supports_moonshot_provider(self):
    client = create_llm_client("moonshot", "kimi-k2.5")
    self.assertIsInstance(client, OpenAIClient)
    self.assertEqual(client.provider, "moonshot")
```

- [ ] **Step 2: Run tests to verify Moonshot support fails before implementation**

Run: `python3 -m unittest tests.test_model_validation -v`
Expected: FAIL because `create_llm_client("moonshot", ...)` raises `ValueError` and the Moonshot model is missing from the catalog.

- [ ] **Step 3: Add minimal provider wiring**

```python
_PROVIDER_CONFIG = {
    "xai": ("https://api.x.ai/v1", "XAI_API_KEY"),
    "openrouter": ("https://openrouter.ai/api/v1", "OPENROUTER_API_KEY"),
    "ollama": ("http://localhost:11434/v1", None),
    "moonshot": ("https://api.moonshot.cn/v1", "MOONSHOT_API_KEY"),
}
```

```python
if provider_lower in ("openai", "ollama", "openrouter", "moonshot"):
    return OpenAIClient(model, base_url, provider=provider_lower, **kwargs)
```

- [ ] **Step 4: Run the provider tests to verify they pass**

Run: `python3 -m unittest tests.test_model_validation -v`
Expected: PASS for the new Moonshot validation and factory coverage.

- [ ] **Step 5: Commit**

```bash
git add tests/test_model_validation.py tradingagents/llm_clients/openai_client.py tradingagents/llm_clients/factory.py
git commit -m "feat: wire moonshot provider support"
```

### Task 2: Expose Moonshot In CLI Model Selection

**Files:**
- Modify: `tradingagents/llm_clients/model_catalog.py`
- Modify: `cli/utils.py`
- Test: `tests/test_model_validation.py`

- [ ] **Step 1: Extend the catalog with the first-release Kimi model**

```python
"moonshot": {
    "quick": [
        ("Kimi K2.5 - Recommended first-release Moonshot model", "kimi-k2.5"),
    ],
    "deep": [
        ("Kimi K2.5 - Recommended first-release Moonshot model", "kimi-k2.5"),
    ],
},
```

- [ ] **Step 2: Add Moonshot to the CLI provider list**

```python
BASE_URLS = [
    ("OpenAI", "https://api.openai.com/v1"),
    ("Google", None),
    ("Anthropic", "https://api.anthropic.com/"),
    ("xAI", "https://api.x.ai/v1"),
    ("Moonshot / Kimi", "https://api.moonshot.cn/v1"),
    ("Openrouter", "https://openrouter.ai/api/v1"),
    ("Ollama", "http://localhost:11434/v1"),
]
```

- [ ] **Step 3: Run tests to verify catalog-backed validation passes**

Run: `python3 -m unittest tests.test_model_validation -v`
Expected: PASS, including Moonshot model validation through the shared catalog.

- [ ] **Step 4: Commit**

```bash
git add tradingagents/llm_clients/model_catalog.py cli/utils.py tests/test_model_validation.py
git commit -m "feat: expose moonshot provider in cli"
```

### Task 3: Document And Verify The End-To-End Path

**Files:**
- Modify: `README.md`
- Test: `tests/test_model_validation.py`

- [ ] **Step 1: Add README configuration guidance for Moonshot**

```md
export MOONSHOT_API_KEY=...        # Moonshot (Kimi)
```

```python
config["llm_provider"] = "moonshot"
config["deep_think_llm"] = "kimi-k2.5"
config["quick_think_llm"] = "kimi-k2.5"
```

- [ ] **Step 2: Run the targeted test suite**

Run: `python3 -m unittest tests.test_model_validation -v`
Expected: PASS with Moonshot included in the provider and model validation path.

- [ ] **Step 3: Validate the OpenSpec change**

Run: `openspec validate support-kimi-provider`
Expected: `Change 'support-kimi-provider' is valid`

- [ ] **Step 4: Commit**

```bash
git add README.md openspec/changes/support-kimi-provider docs/superpowers/plans/2026-04-09-support-kimi-provider.md
git commit -m "docs: add kimi provider rollout plan"
```
