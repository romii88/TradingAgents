# Kimi / 智谱 接入设计说明

## 1. 目标

在尽量不改动现有 agent 编排逻辑的前提下，让 TradingAgents 能够像选择 OpenAI、Anthropic、Google 一样，正式支持：

- Kimi（Moonshot）
- 智谱（GLM / BigModel）

目标不仅是“底层 API 勉强能调用”，而是：

- Python 配置可直接使用
- CLI 可直接选择 provider 和模型
- README 中有明确配置方法
- 至少具备基础测试覆盖

---

## 2. 现状判断

### 2.1 好消息

当前仓库已经有一套较清晰的 provider 抽象：

- `factory.py` 负责 provider 分发
- `OpenAIClient` 已经统一兼容了多种 OpenAI-like provider
- `TradingAgentsGraph` 只依赖 provider 名称和模型名

这意味着新增 Kimi / 智谱不需要侵入 agent 逻辑。

### 2.2 真正缺的部分

当前还缺以下产品化支持：

- CLI 没有 Kimi / 智谱 provider 入口
- `model_catalog.py` 没有这两个 provider 的模型列表
- `OpenAIClient` 没有这两个 provider 的默认 base URL / 环境变量映射
- README 没有写如何配置 Kimi / 智谱 API Key
- 测试没有覆盖这两个 provider

---

## 3. 两种可选接入方式

## 方案 A：只把它们当作 OpenAI-compatible endpoint

做法：

- `llm_provider` 仍使用 `openai`
- 用户手动把 `backend_url` 改成 Kimi 或智谱地址
- 模型名直接填 `moonshot-*` 或 `glm-*`

优点：

- 改动最少
- 几乎不需要改 factory 结构

缺点：

- CLI 体验差
- 文档可读性差
- provider 语义混乱
- 后续很难为不同 provider 做独立的默认配置和测试

结论：

适合作为临时兼容手段，不适合作为正式二开方案。

---

## 方案 B：把 Kimi / 智谱作为正式 provider

做法：

- 新增 provider 名称：`moonshot`、`zhipu`
- 仍然复用 `OpenAIClient`
- 在 `_PROVIDER_CONFIG` 里增加默认 `base_url` 与环境变量
- 在 CLI 中增加 provider 选项
- 在 model catalog 中增加对应模型选项

优点：

- 最符合当前架构
- 对业务层零侵入
- CLI、文档、测试都更清晰
- 以后再扩展 DeepSeek、百川、阿里通义也能沿用同一路径

缺点：

- 需要同步改 5 到 7 个文件

结论：

这是最推荐的方案。

---

## 方案 C：为每个 provider 单独写 client

做法：

- 新建 `moonshot_client.py`
- 新建 `zhipu_client.py`
- factory 单独分发

优点：

- 如果以后它们出现与 OpenAI API 明显不兼容的特性，扩展空间更大

缺点：

- 当前阶段明显过度设计
- 大量逻辑会和 `OpenAIClient` 重复

结论：

除非后续明确需要 provider 特有参数，否则现在不建议。

---

## 4. 推荐方案

推荐采用 **方案 B：正式 provider + 复用 `OpenAIClient`**。

原因很直接：

- 当前架构已经证明 OpenAI-compatible provider 可以走统一 client
- Kimi 与智谱的首期接入需求主要是“能选、能配、能跑”
- 这样改动最小，收益最大

---

## 5. 推荐改动范围

## 5.1 `tradingagents/llm_clients/openai_client.py`

目标：

- 为 `moonshot`、`zhipu` 增加默认 endpoint 和 API key 环境变量映射

预期改动：

- 在 `_PROVIDER_CONFIG` 中新增：
  - `moonshot`: 默认 base URL + `MOONSHOT_API_KEY`
  - `zhipu`: 默认 base URL + `ZHIPU_API_KEY`

影响：

- Python 代码可以直接通过 provider 名称使用
- 不必要求用户显式填写 base URL

## 5.2 `tradingagents/llm_clients/factory.py`

目标：

- 将 `moonshot` 与 `zhipu` 分发给 `OpenAIClient`

影响：

- graph 层无需知道它们的具体实现

## 5.3 `tradingagents/llm_clients/model_catalog.py`

目标：

- 为两个 provider 增加 quick/deep 默认模型列表

影响：

- CLI 能展示可选模型
- validator 可复用 catalog

## 5.4 `cli/utils.py`

目标：

- 在 provider 列表中增加：
  - Moonshot / Kimi
  - Zhipu / GLM

影响：

- 用户能在 CLI 里正式选择

## 5.5 `tradingagents/default_config.py`

目标：

- 可选地调整注释，说明支持更多 OpenAI-compatible provider

影响：

- 配置语义更清晰

## 5.6 `README.md`

目标：

- 增加环境变量说明
- 增加 provider 支持说明
- 增加示例配置

## 5.7 `tests/`

目标：

- 覆盖新 provider 的模型校验和工厂行为

---

## 6. 为什么不用改 agent 和 graph

因为当前业务层只依赖两个抽象：

- `llm.invoke(...)`
- `llm.bind_tools(...)`

而这些能力已经被 provider client 封装好了。只要 Kimi / 智谱在 LangChain 的 `ChatOpenAI` 路径下可以正常工作，agent 层完全不需要知道具体 provider。

这也是当前项目最值得保留的架构边界。

---

## 7. 风险点

虽然整体改动不大，但有 4 个风险要提前看清：

### 7.1 Tool calling 兼容性

Analyst 节点依赖工具调用。如果某个 provider 的 OpenAI-compatible 实现对 tools/function calling 兼容性不完整，最先出问题的就是 analyst 阶段。

建议：

- 先用单个 analyst 做 smoke test
- 再跑完整 graph

### 7.2 模型命名迭代较快

Kimi / 智谱模型名可能更新较快，静态 catalog 容易过时。

建议：

- 首期提供常用官方模型
- validator 对这类 provider 可以考虑“已知模型优先，但未知模型只告警不阻塞”

### 7.3 provider 特有参数

如果以后 Kimi / 智谱需要特殊参数，当前统一 `OpenAIClient` 可能不够。

建议：

- 第一版不提前设计
- 只有出现真实需求时再拆独立 client

### 7.4 国内网络与代理问题

用户可能需要代理或自建转发层。

建议：

- 保留 `backend_url` 覆盖能力
- 保持 `http_client` / `http_async_client` 透传能力

---

## 8. 实施优先级建议

### P1：最小可用支持

- provider 注册
- CLI provider 入口
- model catalog
- README 配置示例

### P2：稳定性补齐

- validator 调整
- 基础测试
- 示例代码更新

### P3：体验增强

- provider 专属提示文案
- 自定义模型输入
- provider 兼容性说明

---

## 9. 我对这个二开方向的建议

如果你现在是要基于这个开源项目做自己的版本，我建议第一步不要急着大改业务流程，而是先把“模型接入能力”做成稳定底座。  
原因是：

- 这是当前架构里边界最清楚、收益最快的改造点
- 改完以后，你可以立刻开始用更适合中文场景的模型跑完整链路
- 后续再决定是否深入改 prompt、数据源、A 股场景，会更从容

因此，一条比较稳的路径是：

1. 先补架构文档
2. 先接入 Kimi / 智谱
3. 跑通中文输出和几个典型 ticker
4. 再决定是否引入国内金融数据源和策略改造

---

## 10. 下一步建议

下一步我建议直接进入实现前设计确认，只需要定一件事：

- 你是希望把 Kimi / 智谱做成 **CLI 可直接选择的正式 provider**
- 还是先做成 **代码层可配置的兼容 provider**

如果没有特殊限制，我建议直接做“正式 provider”，这样后续维护最省心。
