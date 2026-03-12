# ETF轮动策略 - 技术设计文档

## Context

### 背景

`analysis.md` 已经明确，本次变更的目标不是再做一套“策略模块 + 几个 Python 工具函数”，而是构建一个**由 Agent Skills 能力定义驱动**的 ETF 轮动策略框架。

现有 InStock 已经具备以下基础设施，可作为 skill 的底层执行资源：

- ETF 行情与历史数据获取
- 技术指标计算与回测基础能力
- 交易执行框架与事件调度
- MySQL 持久化能力

因此本次设计的重点不是“再封装一层 Python 模块”，而是把这些现有能力组织成：

1. **可声明的 Skill 能力契约**
2. **可编排的 Skill Registry / Runtime**
3. **受策略规则与权限约束的 Agent 决策流程**

### 当前设计偏差

上一版设计将 `skill` 直接落成 `instock/skills/*.py`，存在以下问题：

- 把“能力定义”误写成“代码模块目录”
- 让 orchestrator 依赖具体模块实现，而不是依赖能力契约
- 无法明确 skill 的输入/输出 schema、前置条件、权限边界、可审计性
- 不利于后续接入不同 executor（Python 函数、规则引擎、外部 Agent、人工确认）

本次重写将修正这一点。

---

## Goals / Non-Goals

### Goals

1. **把 ETF 轮动框架设计为 skills-first 架构**
   - skill 是 agent 可调用的能力单元
   - skill 有独立 contract / schema / policy / side effects 定义
   - skill 可以被 orchestrator 选择、编排、审计

2. **建立清晰的运行时分层**
   - Skill Definition Layer：能力定义
   - Skill Runtime Layer：注册、编排、上下文、审计
   - Executor / Adapter Layer：复用现有 Python 基础设施

3. **保证核心交易逻辑可回测、可复现**
   - 动量排序、趋势过滤、风控约束、组合构建必须保持确定性
   - AI 类 skill 只做解释、补充判断、建议增强，不允许绕过核心约束

4. **把 live / backtest 的边界设计清楚**
   - 回测优先运行确定性 skill 图
   - AI skill 在回测中默认禁用、替换为 no-op、或使用录制结果回放

5. **为后续实现保留足够灵活性**
   - Python 只是 executor 之一，不是 skill 的定义形式本身

### Non-Goals

1. 不把 skill 等同于独立 Python 模块目录
2. 不让 AI 直接绕过风控或直接下单
3. 不在本次设计中引入新的重型外部基础设施
4. 不改造现有全站数据源与交易框架的底层协议

---

## Decisions

### Decision 1: Skill 是能力契约，不是 Python 模块

**决定**：本项目中的 skill 首先是一个**声明式能力定义**，包含能力意图、输入输出、约束、权限、执行绑定与审计要求；底层可以映射到 Python executor，但两者不是同一概念。

**一个 skill 至少应定义以下字段：**

- `skill_id`
- `purpose`
- `mode`：`deterministic` / `ai_assisted`
- `triggers`：scheduled / event / manual
- `required_context`
- `input_schema`
- `output_schema`
- `preconditions`
- `side_effects`
- `permissions`
- `fallback_policy`
- `executor_binding`
- `observability`

**理由**：

- agent 编排依赖能力语义，而不是依赖某个 Python 文件路径
- 后续可以替换 executor，而无需重写 skill contract
- 更适合 OpenSpec 的能力建模与验收

**替代方案**：

- ❌ `instock/skills/*.py` 直接代表 skill
- ❌ orchestrator 直接 import 各类模块函数并硬编码顺序

---

### Decision 2: Orchestrator 依赖 Skill Registry，而不是依赖具体实现

**决定**：轮动框架的 orchestrator 只面向 `skill registry` 工作，按当前策略上下文、规则配置和风险状态选择下一步应执行的 skill。

**核心职责**：

- 读取策略 policy 与 skill manifest
- 维护本轮决策的 execution context
- 基于 preconditions 选择可用 skill
- 按阶段编排 skill 执行顺序
- 合并 skill 输出为 `decision envelope`
- 对有副作用的 skill 执行审批 / 拦截 / 审计

**理由**：

- 让“策略框架”真正由 skills 驱动
- skill 可以扩展，但编排中心不必跟着大量改代码

---

### Decision 3: Skills 通过共享 Context 通信，而不是直接互相调用

**决定**：skill 之间不直接 import / 调用彼此实现；统一通过 `StrategyContext`、`WorkingMemory` 和结构化 result 交换信息。

**上下文最小集合**：

- `market_snapshot`
- `universe_snapshot`
- `candidate_rankings`
- `risk_state`
- `market_context`
- `target_portfolio`
- `approval_state`
- `execution_trace_refs`

**运行时对象边界**：

- `StrategyContext`：本轮 cycle 的权威状态容器，由 runtime 创建并持有，保存后续 skill 可消费的结构化结果
- `WorkingMemory`：单轮编排中的临时推理区，用于保存中间结果、降级信息、人工复核线索，不作为跨轮持久状态
- `SkillResultEnvelope`：每次 skill 调用的统一返回结构，描述状态、payload、trace ref、side effect 摘要
- `DecisionEnvelope`：orchestrator 聚合后的统一决策对象，是审批、审计、解释、执行的共同输入

**`StrategyContext` 建议字段**：

- `cycle_id`
- `trigger_type`
- `mode`
- `as_of_time`
- `policy_snapshot`
- `universe_snapshot`
- `portfolio_snapshot`
- `market_snapshot`
- `candidate_rankings`
- `risk_state`
- `market_context`
- `target_portfolio`
- `trade_proposal`
- `approval_state`
- `execution_trace_refs`
- `final_decision_ref`

**`WorkingMemory` 建议字段**：

- `stage_notes`
- `fallback_events`
- `retry_counters`
- `human_review_queue`
- `advisory_annotations`
- `transient_candidates`

**所有权与写入规则**：

- runtime/orchestrator 负责创建、冻结和提交 `StrategyContext`
- deterministic skill 只能写入其 manifest 声明允许写入的上下文字段
- AI-assisted skill 默认只允许写入 `WorkingMemory.advisory_annotations`、`human_review_queue` 和解释性字段
- skill 不允许直接覆盖其他 skill 已提交的权威结果；如需修正，必须写入新的 result envelope，由 orchestrator 按优先级合并
- `policy_snapshot`、`universe_snapshot`、`mode`、`cycle_id` 一经创建即只读
- `execution_trace` 与 `approval_state.history` 只能 append，不允许原地改写

**理由**：

- 避免 skill 之间形成隐式依赖和循环依赖
- 方便审计“某个决策是由哪些 skill 输出共同形成的”

---

### Decision 4: Skill 分为确定性技能与 AI 辅助技能

**决定**：本框架明确区分两类 skill：

1. **确定性 skill**：回测/实盘必须一致
2. **AI 辅助 skill**：提供解释、上下文补充、异常提示、人工确认建议

**确定性 skill** 包括：

- `market-data-snapshot`
- `momentum-ranking`
- `risk-guard`
- `portfolio-constructor`
- `rebalance-executor`（回测为模拟执行）

**AI 辅助 skill** 包括：

- `market-context-interpreter`
- `decision-explainer`

**边界约束**：

- AI skill 不得直接修改历史行情
- AI skill 不得绕过 `risk-guard`
- AI skill 不得直接触发真实交易
- 超出 guardrail 的调整必须进入人工确认

---

### Decision 5: Python 代码放在 Executor / Adapter 层

**决定**：具体 Python 实现属于 executor / adapter 层，用于绑定 skill 到现有系统能力，例如：

- ETF 数据 adapter → 复用 `fund_etf_em.py`
- 指标 / 动量 adapter → 复用 `calculate_indicator.py` 与现有回测逻辑
- 交易 adapter → 复用 `trade/robot/` 与 easytrader

这层可以是 Python，但 skill 定义本身不依赖“必须是某个 Python 模块”。

**建议目录方向**：

```text
config/etf_rotation/
  policy.yaml
  universe.yaml
  skills/
    market-data-snapshot.yaml
    momentum-ranking.yaml
    risk-guard.yaml
    market-context-interpreter.yaml
    portfolio-constructor.yaml
    rebalance-executor.yaml

instock/etf_rotation/runtime/
  registry.py
  orchestrator.py
  context.py
  approval.py
  audit.py

instock/etf_rotation/executors/
  market_data_executor.py
  momentum_executor.py
  risk_executor.py
  portfolio_executor.py
  trade_executor.py
  explain_executor.py

instock/etf_rotation/adapters/
  eastmoney_adapter.py
  indicator_adapter.py
  trade_robot_adapter.py
```

> 上述目录仅表达职责边界，不要求先创建一个“skills 代码目录”来代表 skill 本身。

---

### Decision 6: 真实交易必须经过权限与审批闸门

**决定**：所有带 side effect 的 skill 都必须声明权限等级；其中 `rebalance-executor` 属于高风险技能，live 模式必须经过 approval gate。

**建议权限级别**：

- `read_only`
- `advisory`
- `state_write`
- `trade_proposal`
- `trade_execute`

**审批规则**：

- backtest：自动通过模拟执行
- paper/live proposal：可自动生成交易计划
- live execute：必须满足风控闸门 + 人工确认策略

**状态语义**：

- `trade_proposal`：允许生成交易计划，但不能直接触发真实下单
- `trade_execute`：允许把已批准 proposal 转成真实执行动作
- permission 是 skill 的静态上限，是否真正执行由 mode、policy、approval gate 共同决定

**ApprovalState 建议状态机**：

- `not_required`
- `proposal_generated`
- `pending_review`
- `approved`
- `rejected`
- `expired`
- `executed`

**执行语义**：

- 默认审批粒度为整轮 `trade_proposal` 审批；是否允许单笔审批由 policy 显式开启
- `rejected` 后不得复用原 proposal 直接重试，必须重新生成 `trade_proposal`
- `approved` 必须带 `approval_token`、`approved_at`、`expires_at`
- live execute 必须校验 `cycle_id + proposal_hash + approval_token` 组成的幂等键
- 执行失败可按 policy 进入 retry 分支，但不得越过审批有效期

---

## Architecture

### 1. 分层架构

```text
Policy Layer
  ├─ strategy policy
  ├─ ETF universe
  └─ guardrails / approval rules

Skill Definition Layer
  ├─ skill manifests
  ├─ input/output schema
  ├─ preconditions / side effects
  └─ executor bindings

Skill Runtime Layer
  ├─ registry
  ├─ orchestrator
  ├─ strategy context
  ├─ working memory
  ├─ audit / trace
  └─ approval gate

Executor / Adapter Layer
  ├─ market data executors
  ├─ momentum / risk executors
  ├─ portfolio construction executor
  ├─ trade executor
  └─ AI reasoning executor

Existing Infrastructure Layer
  ├─ fund_etf_em.py
  ├─ calculate_indicator.py
  ├─ backtest tools
  ├─ trade/robot/
  └─ MySQL
```

### 2. 核心 Skill 集

#### 2.1 `market-data-snapshot`

用途：为当前轮动周期生成标准化市场快照。

- 输入：ETF universe、日期/时间窗口、运行模式
- 输出：标准化行情、K 线、流动性、资产池可用性状态
- side effects：无
- mode：deterministic

#### 2.2 `momentum-ranking`

用途：基于确定性规则生成动量排序结果。

- 输入：市场快照、动量参数、趋势过滤参数
- 输出：候选 ETF 排名、排除原因、置信标签
- side effects：可选写入信号记录
- mode：deterministic

#### 2.3 `risk-guard`

用途：在组合级和标的级施加风险约束。

- 输入：当前持仓、组合净值、波动率、冷却状态、候选排名
- 输出：风险状态、禁买列表、减仓建议、风险事件
- side effects：可选写入风险事件
- mode：deterministic

#### 2.4 `market-context-interpreter`

用途：读取新闻/政策/宏观上下文，生成**建议性解释**与风险提示。

- 输入：外部上下文、市场快照、当前候选列表
- 输出：背景解释、异常提示、建议性约束
- side effects：记录 reasoning log
- mode：ai_assisted

#### 2.5 `portfolio-constructor`

用途：在候选列表和风险约束内生成目标组合。

- 输入：候选排名、风险约束、仓位规则、现金保留规则
- 输出：`target_portfolio`
- side effects：无
- mode：deterministic

#### 2.6 `rebalance-executor`

用途：把目标组合转成可执行计划，并在允许时执行。

- 输入：目标组合、当前持仓、交易窗口、审批状态
- 输出：交易计划、执行结果、审计记录
- side effects：真实/模拟交易
- mode：deterministic（执行模式由 runtime 决定）

#### 2.7 `decision-explainer`

用途：把本轮 skill 输出整合成人类可读的决策说明。

- 输入：execution trace、target portfolio、风险结果
- 输出：决策摘要、理由、人工确认提示
- side effects：记录报告
- mode：ai_assisted 或 template_based

### 3. Skill Manifest 示例

```yaml
skill_id: momentum-ranking
purpose: Generate deterministic ETF momentum ranking for rebalance cycle
mode: deterministic
triggers: [scheduled_rebalance, manual_rebalance, backtest_replay]
required_context:
  - market_snapshot
  - strategy_policy.signal_rules
input_schema: MomentumRankingInput
output_schema: MomentumRankingOutput
preconditions:
  - market_snapshot.ready == true
side_effects:
  - type: state_write
    optional: true
    target: signal_log
permissions: state_write
fallback_policy:
  on_error: fail_cycle
executor_binding:
  executor: python
  target: momentum_executor.generate_rankings
observability:
  log_input_digest: true
  log_output_digest: true
  emit_metrics: true
```

### 4. 轮动执行流程

#### 4.1 周度调仓流程

1. orchestrator 创建新一轮 `StrategyContext`
2. 调用 `market-data-snapshot`
3. 调用 `momentum-ranking`
4. 调用 `risk-guard`
5. 可选调用 `market-context-interpreter`
6. 调用 `portfolio-constructor`
7. 生成 `decision envelope`
8. 调用 `decision-explainer`
9. 若满足权限与审批条件，调用 `rebalance-executor`
10. 记录完整 execution trace

#### 4.2 日内监控流程

1. 周期性刷新轻量 market snapshot
2. 只执行 `risk-guard`
3. 若触发风险动作，生成 `trade proposal`
4. 经 approval gate 后再进入 `rebalance-executor`

### 5. Runtime 核心对象

#### 5.1 `SkillResultEnvelope`

每次 skill 执行都必须返回统一 envelope，建议至少包含：

- `skill_id`
- `invocation_id`
- `cycle_id`
- `status`：`success` / `degraded` / `failed` / `skipped` / `no_op`
- `input_digest`
- `output_payload`
- `written_context_keys`
- `errors`
- `warnings`
- `side_effect_summary`
- `trace_ref`
- `started_at` / `finished_at` / `latency_ms`

#### 5.2 `DecisionEnvelope`

orchestrator 每轮输出统一的 `decision envelope`，至少包含：

- `cycle_id`
- `decision_id`
- `trigger_type`
- `mode`
- `as_of_time`
- `policy_version`
- `selected_skills`
- `skill_results`
- `candidate_rankings`
- `risk_constraints`
- `risk_events`
- `market_advisory`
- `target_portfolio`
- `trade_proposal`
- `approval_state`
- `idempotency_key`
- `fallback_summary`
- `human_readable_rationale`
- `audit_refs`

**聚合规则**：

- `candidate_rankings` 只能来自确定性 ranking 结果
- `risk_constraints` 和 `risk_events` 只能来自 `risk-guard` 与 policy guardrail
- `market_advisory` 只作为解释、提示、人工复核输入，不直接改写目标组合
- `target_portfolio` 必须记录 ranking 来源、风险约束版本与构建理由
- `trade_proposal` 只能由 `rebalance-executor` 的 proposal 阶段生成

#### 5.3 失败、降级与重试策略

- `fail-fast`：`market-data-snapshot`、`momentum-ranking`、`risk-guard` 任一关键 skill 失败时终止本轮
- `degrade`：AI-assisted skill 失败时允许降级为无 AI 注记或模板化解释
- `retry`：只允许对声明为 retryable 的外部 IO 动作进行有限重试
- `human-review`：当 AI 输出越界、审批冲突、风险状态不一致时，转人工复核

### 6. 运行模式矩阵

| 组件 | Backtest | Paper | Live |
|------|----------|-------|------|
| market-data-snapshot | 历史回放 | 当日/近实时数据，不写真实侧效 | 实时/当日数据 |
| momentum-ranking | 启用 | 启用 | 启用 |
| risk-guard | 启用 | 启用 | 启用 |
| market-context-interpreter | 默认禁用 / no-op / 录制回放 | 可启用，仅 advisory | 可启用，仅 advisory |
| portfolio-constructor | 启用 | 启用 | 启用 |
| decision-explainer | 可选模板化 | 启用 | 启用 |
| rebalance-executor: proposal | 模拟生成 | 生成 proposal | 生成 proposal |
| rebalance-executor: execute | 模拟成交回执 | 仅模拟回执，不触达券商 | 审批通过后真实执行 |

**模式定义**：

- `backtest`：以历史窗口重放为准，要求确定性、可复现；AI 默认禁用、no-op 或使用已录制结果
- `paper`：允许走完整 runtime、生成 proposal、产生模拟执行回执，但不得调用真实券商下单接口
- `live`：允许真实执行，但只有在风控、审批、交易窗口全部通过后才可进入 execute

设计要求：**AI skill 不得成为回测结果的决定性依赖项；`paper` 不得被实现成“偷偷调用真实交易接口”。**

### 7. AI Advisory 边界

AI-assisted skill 允许写入的内容仅限：

- `WorkingMemory.advisory_annotations`
- `WorkingMemory.human_review_queue`
- `DecisionEnvelope.market_advisory`
- `human_readable_rationale`

AI-assisted skill 明确禁止直接写入或改写：

- `candidate_rankings`
- `risk_constraints`
- `risk_events`
- `target_portfolio`
- `trade_proposal`
- `approval_state`

orchestrator 不得自动把 AI 文本建议提升为硬约束；若策略需要让 AI 建议影响交易，只能通过以下路径之一：

1. 转换为人工复核事项，由人工确认后再进入审批流
2. 转换为新的 policy 变更，由下一轮 cycle 以确定性规则生效

---

## Risks / Trade-offs

### Risk 1: Skill 定义过于抽象，落地困难

**风险**：如果只写概念，不定义 schema 和 binding，最终仍会退回“手写模块函数”模式。

**缓解**：

- 每个 skill 必须有 manifest
- manifest 必须可校验
- executor binding 必须明确但不等于 skill 本身

### Risk 2: AI 建议侵入确定性主路径

**风险**：AI skill 如果直接改变仓位决策，会破坏回测一致性。

**缓解**：

- AI skill 只允许输出 advisory annotation、review reason 或 rationale
- 真正生效的硬约束只能来自 policy、risk-guard、approval gate

### Risk 3: Runtime 复杂度上升

**风险**：引入 registry / context / audit 后，短期实现成本高于直接写模块。

**缓解**：

- 第一阶段只实现最小 skill runtime
- 优先支持 ETF 轮动一条主链路
- 复用现有基础设施，不重复造轮子

### Risk 4: Skill 间数据口径不一致

**风险**：不同 executor 生成的字段口径不一致，导致 orchestrator 难以组合。

**缓解**：

- 统一 `market snapshot` / `ranking result` / `risk state` / `target portfolio` schema
- 在 registry 加 schema validation

---

## Migration Plan

### 阶段 1：定义能力模型

1. 定义 skill manifest schema
2. 定义共享 context schema
3. 定义 side effects / permission / approval taxonomy

### 阶段 2：建立最小 runtime

1. 实现 registry
2. 实现 orchestrator
3. 实现 execution trace 与审计日志

### 阶段 3：绑定确定性 skills

1. 接入 market-data-snapshot
2. 接入 momentum-ranking
3. 接入 risk-guard
4. 接入 portfolio-constructor
5. 接入 backtest / simulated rebalance-executor

### 阶段 4：接入 AI 辅助 skills

1. 接入 market-context-interpreter
2. 接入 decision-explainer
3. 加入人工确认与风控闸门

### 回滚策略

- 可关闭 agent runtime，退回纯确定性轮动流程
- 可单独禁用 AI 辅助 skills
- trade execute 可降级为 proposal only

---

## Open Questions

1. `market-context-interpreter` 的外部数据来源范围到哪里为止？仅财经新闻，还是包含政策公告与宏观数据库？
2. 是否需要支持“整轮审批 + 单笔例外审批”的混合审批模型，还是先只支持整轮 proposal 审批？
3. skill manifest 应放在策略配置目录下，还是单独作为 agent runtime 配置目录管理？

---

> 依赖文档: [proposal.md](./proposal.md)
>
> `proposal.md` 与 `specs/` 应继续与本设计保持同步，避免回退到模块化叙事。