# TradingAgents 项目架构说明

## 1. 项目目标

TradingAgents 是一个基于 LangGraph 的多智能体金融分析框架。它把一次交易决策拆成多类角色协作完成：

- Analyst Team：生成市场、情绪、新闻、基本面四类分析报告
- Research Team：围绕是否值得投资展开多轮多空辩论
- Trader：基于研究结论形成交易计划
- Risk Management / Portfolio Manager：围绕风险进行讨论并输出最终评级

对二次开发来说，这个仓库最大的特点是：

- 业务流程主要集中在 `graph` 和 `agents`
- 模型接入主要集中在 `llm_clients`
- 行情/新闻/财报数据主要集中在 `dataflows`
- CLI 只是一个交互外壳，不是核心业务层

---

## 2. 顶层目录职责

### `tradingagents/graph`

系统编排层，负责把各类 agent 拼成一个完整执行图。

- `trading_graph.py`
  - 总入口类 `TradingAgentsGraph`
  - 初始化配置、模型客户端、memory、tool nodes、graph
  - 对外暴露 `propagate()` 作为一次完整分析调用入口
- `setup.py`
  - 用 `StateGraph` 组装 analyst -> researcher -> trader -> risk manager 的节点与边
- `conditional_logic.py`
  - 定义 analyst 工具循环、多空辩论轮次、风险讨论轮次
- `propagation.py`
  - 构造初始状态和调用参数
- `signal_processing.py`
  - 从最终文本中提取更结构化的交易信号
- `reflection.py`
  - 根据收益/亏损结果做反思与记忆更新

### `tradingagents/agents`

业务角色层，每个文件对应一个 agent 的 prompt 和状态写入逻辑。

- `analysts/`
  - 市场、社媒、新闻、基本面分析师
- `researchers/`
  - 多头研究员、空头研究员
- `managers/`
  - `research_manager.py` 负责投资辩论裁决
  - `portfolio_manager.py` 负责最终评级和交易决策
- `trader/`
  - 将各类报告整合成交易执行建议
- `risk_mgmt/`
  - 激进、中性、保守三类风险角色
- `utils/`
  - agent 状态定义、工具封装、记忆、语言输出等通用能力

### `tradingagents/llm_clients`

模型适配层，是二次开发支持新模型的核心区域。

- `factory.py`
  - 按 provider 名称创建具体 LLM client
- `base_client.py`
  - 提供统一抽象和响应内容归一化逻辑
- `openai_client.py`
  - 当前兼容 `openai`、`xai`、`openrouter`、`ollama`
- `google_client.py`
  - Gemini 适配
- `anthropic_client.py`
  - Claude 适配
- `model_catalog.py`
  - CLI 使用的模型清单
- `validators.py`
  - provider/model 名称校验

### `tradingagents/dataflows`

金融数据访问层，当前重点支持 `yfinance` 与 `alpha_vantage`。

- `interface.py`
  - 根据类别或工具名把调用路由到具体 vendor
- `config.py`
  - 维护当前运行态配置
- 其他文件
  - 按数据源实现行情、指标、新闻、财务数据获取

### `cli`

终端交互界面层。

- `main.py`
  - 交互式收集参数、启动 graph、展示运行过程、落盘报告
- `utils.py`
  - provider/model 选择器、语言选择器、ticker/date 输入等
- `stats_handler.py`
  - 收集 token / tool 调用统计

---

## 3. 核心运行链路

一次完整分析的大致流程如下：

1. CLI 或 `main.py` 构造配置
2. `TradingAgentsGraph` 读取配置并调用 `create_llm_client()`
3. 创建 quick / deep 两类模型实例
4. 初始化 memory、tool nodes、conditional logic
5. `GraphSetup.setup_graph()` 组装 LangGraph 工作流
6. `propagate()` 创建初始状态并执行 graph
7. 各 analyst 按顺序执行，必要时调用工具获取数据
8. 多空研究员围绕 analyst 报告进行多轮辩论
9. Research Manager 形成投资方案
10. Trader 形成交易计划
11. 风险角色展开多轮风险辩论
12. Portfolio Manager 输出最终评级与决策
13. `process_signal()` 从最终文本提炼核心信号
14. `trading_graph.py` 将结果写入 `results/`

可以把它理解成 4 层流水线：

- 输入配置层：CLI / `DEFAULT_CONFIG`
- 模型与数据层：`llm_clients` + `dataflows`
- 业务角色层：`agents`
- 编排执行层：`graph`

---

## 4. 配置系统如何工作

默认配置在 `tradingagents/default_config.py` 中，关键字段包括：

- `llm_provider`
- `deep_think_llm`
- `quick_think_llm`
- `backend_url`
- `google_thinking_level`
- `openai_reasoning_effort`
- `anthropic_effort`
- `output_language`
- `data_vendors`
- `tool_vendors`

配置注入路径是：

1. CLI 收集用户选择
2. 组装到 `config`
3. `TradingAgentsGraph.__init__()` 调用 `set_config(config)`
4. `dataflows` 和 `agents` 在运行时通过 `get_config()` 读取

这里的好处是：

- agent prompt 可以读取语言配置
- dataflow 可以读取数据源配置
- graph 可以读取 provider/model 配置

这也是二开时最值得复用的一点：配置入口统一，扩展成本低。

---

## 5. 模型接入架构

### 5.1 当前实现方式

模型接入不是直接散落在 agent 中，而是集中在 `llm_clients`：

1. `TradingAgentsGraph` 根据 `llm_provider` 调用 `create_llm_client()`
2. factory 选择具体 provider client
3. client 返回 LangChain 兼容的 chat model 实例
4. agent 只依赖 `llm.invoke(...)` 或 `llm.bind_tools(...)`

这意味着 agent 层对 provider 基本无感知。

### 5.2 为什么这对二次开发很友好

只要新的模型提供商满足以下条件，就可以低侵入接入：

- 能返回 LangChain 可用的 chat model
- 能兼容当前的普通对话调用
- analyst 场景最好支持 tool calling

因此新增 Kimi 或智谱时，理论上不需要修改：

- `tradingagents/agents/*`
- `tradingagents/graph/setup.py`
- `tradingagents/graph/conditional_logic.py`

主要改动会落在：

- `tradingagents/llm_clients/*`
- `cli/utils.py`
- `tradingagents/default_config.py`
- `README.md`
- 测试文件

---

## 6. Agent 层设计特点

### 6.1 Analyst 类角色

以 `market_analyst.py` 为代表，模式基本一致：

- 构造 prompt
- 绑定对应工具
- 执行 `chain.invoke(state["messages"])`
- 如果本轮没有 tool call，则把最终文本写回对应 report 字段

这类节点高度依赖 tool calling 能力，所以如果某个 provider 的工具调用兼容性较差，会先影响 analyst 阶段。

### 6.2 辩论类角色

多头/空头研究员与风险讨论角色大多是：

- 从 state 中读取上游报告和历史对话
- 拼接 prompt
- 直接 `llm.invoke(prompt)`
- 把输出写回 debate state

这类角色对 provider 的要求相对更低，只要基础文本生成稳定即可。

### 6.3 Manager / Trader 类角色

这些角色负责总结、裁决和结构化输出，是结果质量最敏感的节点：

- `research_manager.py`
- `trader/trader.py`
- `managers/portfolio_manager.py`

如果后续你要做更稳定的结构化输出，优先考虑从这些角色入手做 schema 化输出或结果校验。

---

## 7. 数据层设计特点

数据层通过 `dataflows/interface.py` 做了一层路由抽象：

- 工具名 -> 数据类别
- 数据类别 / 工具名 -> vendor
- vendor -> 具体实现

已有两个优点：

- 可以统一配置默认 vendor
- 某个工具可以做 tool-level override

这让后续扩展数据源也比较自然，比如：

- AkShare
- Tushare
- 聚宽
- 东方财富/同花顺类数据接口

如果你后续要做 A 股/港股/中概更深度支持，这个层会是第二个重点改造区。

---

## 8. 当前代码里与二次开发最相关的结论

### 8.1 架构上已经具备“多 provider 扩展点”

这是当前项目最适合二开的地方。模型接入和 agent 编排是分离的，说明：

- 新增 provider 的边界比较明确
- 对上层业务影响较小
- 更适合先做“支持更多模型”，再逐步做“优化策略/中文市场适配”

### 8.2 当前 CLI 的 provider 是固定枚举

虽然底层 `OpenAIClient` 已经支持自定义 `base_url`，但 CLI 层不支持：

- 自定义 provider
- 自定义 OpenAI-compatible endpoint
- 自定义模型列表

这意味着“代码层面部分可兼容”，但“产品层面还不算真正支持 Kimi/智谱”。

### 8.3 validator 对新 provider 的约束还不完整

当前 `validators.py` 只对已有 provider 做静态校验，对 `ollama/openrouter` 放宽。  
如果接入 Kimi/智谱，需要决定：

- 走严格静态 catalog
- 还是走“OpenAI-compatible provider 默认允许任意模型名”

这个选择会影响后续维护成本和 CLI 体验。

---

## 9. 推荐的二次开发顺序

基于当前代码结构，建议按下面顺序推进：

### 第一阶段：补文档和扩展设计

- 补当前项目架构文档
- 识别模型接入边界
- 明确 Kimi/智谱是作为独立 provider 还是 OpenAI-compatible provider family

### 第二阶段：补模型 provider 能力

- 增加 `moonshot` / `zhipu` provider
- 增加模型 catalog
- 增加 CLI 选择入口
- 增加环境变量与 README 配置说明
- 增加最小测试覆盖

### 第三阶段：做中国市场适配

- 中文默认输出优化
- A 股 / 港股 ticker 体验优化
- 国内金融数据源接入
- 面向中文资讯的数据质量增强

### 第四阶段：做策略与结果增强

- 结构化交易信号
- 回测接口
- 风险参数模板
- 多市场、多账户、多资产类别扩展

---

## 10. 对你当前需求的直接建议

如果你的第一目标是“尽快让项目支持 Kimi 和智谱”，最稳妥的方向不是重写 agent，而是：

1. 保持 `agents` 与 `graph` 不动
2. 在 `llm_clients` 中复用现有 OpenAI-compatible 适配思路
3. 在 CLI 中把新 provider 作为正式选项暴露出来
4. 用文档和测试保证使用者知道怎么配置

换句话说，这个项目已经有一个不错的“模型接入骨架”，我们更像是在补齐 provider 产品化支持，而不是重构整个系统。
