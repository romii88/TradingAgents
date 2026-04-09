# ModelArts Provider Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a first-class `modelarts` provider so TradingAgents can directly use the fixed Huawei Cloud DeepSeek deployment from the CLI and Python config.

**Architecture:** Reuse the existing OpenAI-compatible client path instead of creating a new client class. The change stays inside provider registration, shared model catalog, CLI provider selection, docs, and validation tests, leaving graph and agent orchestration untouched.

**Tech Stack:** Python, LangChain `ChatOpenAI`, Questionary CLI, unittest

---

### Task 1: Register The Provider In The LLM Client Layer

**Files:**
- Modify: `tradingagents/llm_clients/openai_client.py`
- Modify: `tradingagents/llm_clients/factory.py`
- Test: `tests/test_model_validation.py`

- [ ] **Step 1: Write the failing test**

```python
def test_create_llm_client_supports_modelarts_provider(self):
    client = create_llm_client("modelarts", "deepseek-v3.1-terminus")

    self.assertIsInstance(client, OpenAIClient)
    self.assertEqual(client.provider, "modelarts")
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python -m unittest tests.test_model_validation.ModelValidationTests.test_create_llm_client_supports_modelarts_provider -v`
Expected: FAIL with `Unsupported LLM provider: modelarts`

- [ ] **Step 3: Write minimal implementation**

```python
# tradingagents/llm_clients/openai_client.py
_PROVIDER_CONFIG = {
    "xai": ("https://api.x.ai/v1", "XAI_API_KEY"),
    "openrouter": ("https://openrouter.ai/api/v1", "OPENROUTER_API_KEY"),
    "ollama": ("http://localhost:11434/v1", None),
    "moonshot": ("https://api.moonshot.cn/v1", "MOONSHOT_API_KEY"),
    "modelarts": ("https://api.modelarts-maas.com/openai/v1", "MODELARTS_API_KEY"),
}

# tradingagents/llm_clients/factory.py
if provider_lower in ("openai", "ollama", "openrouter", "moonshot", "modelarts"):
    return OpenAIClient(model, base_url, provider=provider_lower, **kwargs)
```

- [ ] **Step 4: Run test to verify it passes**

Run: `python -m unittest tests.test_model_validation.ModelValidationTests.test_create_llm_client_supports_modelarts_provider -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add tradingagents/llm_clients/openai_client.py tradingagents/llm_clients/factory.py tests/test_model_validation.py
git commit -m "feat: add modelarts provider registration"
```

### Task 2: Add Shared Catalog And CLI Provider Selection

**Files:**
- Modify: `tradingagents/llm_clients/model_catalog.py`
- Modify: `cli/utils.py`
- Test: `tests/test_model_validation.py`

- [ ] **Step 1: Write the failing test**

```python
def test_cli_catalog_models_are_all_validator_approved(self):
    for provider, models in get_known_models().items():
        if provider in ("ollama", "openrouter"):
            continue

        for model in models:
            with self.subTest(provider=provider, model=model):
                self.assertTrue(validate_model(provider, model))
```

- [ ] **Step 2: Run test to verify it fails after adding the catalog entry only**

Run: `python -m unittest tests.test_model_validation.ModelValidationTests.test_cli_catalog_models_are_all_validator_approved -v`
Expected: FAIL if the new provider is added inconsistently

- [ ] **Step 3: Write minimal implementation**

```python
# tradingagents/llm_clients/model_catalog.py
"modelarts": {
    "quick": [
        ("DeepSeek V3.1 Terminus - Huawei Cloud fixed deployment", "deepseek-v3.1-terminus"),
    ],
    "deep": [
        ("DeepSeek V3.1 Terminus - Huawei Cloud fixed deployment", "deepseek-v3.1-terminus"),
    ],
},

# cli/utils.py
providers = [
    ("OpenAI", "openai", "https://api.openai.com/v1"),
    ("Google", "google", None),
    ("Anthropic", "anthropic", "https://api.anthropic.com/"),
    ("xAI", "xai", "https://api.x.ai/v1"),
    ("Moonshot / Kimi", "moonshot", "https://api.moonshot.cn/v1"),
    ("Huawei Cloud / DeepSeek", "modelarts", "https://api.modelarts-maas.com/openai/v1"),
    ("Openrouter", "openrouter", "https://openrouter.ai/api/v1"),
    ("Ollama", "ollama", "http://localhost:11434/v1"),
]
```

- [ ] **Step 4: Run tests to verify shared catalog consistency**

Run: `python -m unittest tests.test_model_validation.ModelValidationTests.test_cli_catalog_models_are_all_validator_approved -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add tradingagents/llm_clients/model_catalog.py cli/utils.py tests/test_model_validation.py
git commit -m "feat: expose modelarts provider in cli"
```

### Task 3: Document Environment Setup And Python Usage

**Files:**
- Modify: `README.md`
- Modify: `.env.example`

- [ ] **Step 1: Update the environment variable examples**

```bash
export MODELARTS_API_KEY=...       # Huawei Cloud / DeepSeek
```

```dotenv
MODELARTS_API_KEY=
```

- [ ] **Step 2: Update the supported provider and Python config examples**

```python
config = DEFAULT_CONFIG.copy()
config["llm_provider"] = "modelarts"
config["deep_think_llm"] = "deepseek-v3.1-terminus"
config["quick_think_llm"] = "deepseek-v3.1-terminus"
```

- [ ] **Step 3: Review the README copy for consistency**

Run: `rg -n "modelarts|MODELARTS_API_KEY|deepseek-v3.1-terminus" README.md .env.example`
Expected: All three strings appear in the correct setup sections

- [ ] **Step 4: Commit**

```bash
git add README.md .env.example
git commit -m "docs: add modelarts provider setup"
```

### Task 4: Extend Validation Coverage And Run Final Checks

**Files:**
- Modify: `tests/test_model_validation.py`

- [ ] **Step 1: Add explicit provider regression coverage**

```python
def test_modelarts_catalog_model_is_validator_approved(self):
    self.assertTrue(validate_model("modelarts", "deepseek-v3.1-terminus"))
```

```python
def test_unknown_model_emits_warning_for_strict_provider(self):
    client = DummyLLMClient("modelarts", "not-a-real-modelarts-model")

    with warnings.catch_warnings(record=True) as caught:
        warnings.simplefilter("always")
        client.get_llm()

    self.assertEqual(len(caught), 1)
    self.assertIn("not-a-real-modelarts-model", str(caught[0].message))
    self.assertIn("modelarts", str(caught[0].message))
```

- [ ] **Step 2: Run the test module**

Run: `python -m unittest tests.test_model_validation -v`
Expected: PASS

- [ ] **Step 3: Run a syntax check on touched Python files**

Run: `python -m py_compile tradingagents/llm_clients/openai_client.py tradingagents/llm_clients/factory.py tradingagents/llm_clients/model_catalog.py cli/utils.py tests/test_model_validation.py`
Expected: no output

- [ ] **Step 4: Optional manual smoke**

Run: `MODELARTS_API_KEY=... .venv/bin/python -c "from tradingagents.llm_clients.factory import create_llm_client; print(create_llm_client('modelarts', 'deepseek-v3.1-terminus').get_llm().invoke('Reply with exactly: MODELARTS_OK').content)"`
Expected: `MODELARTS_OK`

- [ ] **Step 5: Commit**

```bash
git add tests/test_model_validation.py
git commit -m "test: cover modelarts provider validation"
```
