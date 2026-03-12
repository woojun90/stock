# ETF轮动策略 - 变更提案

## Why

当前 `analysis.md` 已经明确，用户要的是一个 **由 agent skills 能力契约驱动** 的 ETF 轮动框架，而不是把“skills”实现成一组 Python 模块或函数入口。

上一版 proposal / specs 的主要问题是：

- 把 skill 叙事落成 `instock/skills/*.py` 目录
- 用函数式 API 描述能力，而不是描述 skill contract
- 没有把 registry、runtime、approval、audit 作为一等设计对象
- 让 AI 能力看起来可能直接影响交易主路径，边界不够清楚

本次变更将 proposal 与 specs 统一收敛到 `design.md` 已确认的 **skills-first / capability-contract** 方向。

## What Changes

### 1. 引入 Skill Contract 作为一等抽象

ETF 轮动框架将围绕 skill manifest / contract 组织，而不是围绕 Python 模块组织。每个 skill 至少定义：

- `skill_id`
- `purpose`
- `mode`
- `required_context`
- `input_schema`
- `output_schema`
- `preconditions`
- `side_effects`
- `permissions`
- `fallback_policy`
- `executor_binding`
- `observability`

### 2. 引入 Runtime / Registry / Context 驱动的编排方式

新增的核心运行时概念包括：

- `Skill Registry`
- `StrategyContext`
- `WorkingMemory`
- `decision envelope`
- `approval gate`
- `execution trace` / `decision audit`

orchestrator 将根据 policy、上下文和 preconditions 选择 skill，而不是直接 import 若干实现模块并固定串联调用。

### 3. 定义首批 ETF 轮动 Skills

本次 proposal 覆盖的首批能力包括：

- `market-data-snapshot`
- `momentum-ranking`
- `risk-guard`
- `market-context-interpreter`
- `portfolio-constructor`
- `rebalance-executor`
- `decision-explainer`
- `rotation-orchestrator`

其中：

- `market-data-snapshot`、`momentum-ranking`、`risk-guard`、`portfolio-constructor`、`rebalance-executor` 属于确定性主路径
- `market-context-interpreter`、`decision-explainer` 属于 AI 辅助能力

### 4. 明确 AI 与交易边界

AI-assisted skills 只能：

- 提供解释
- 提供市场背景与异常提示
- 产生 advisory suggestions

AI-assisted skills 不能：

- 直接改写确定性 ranking
- 绕过 `risk-guard`
- 直接触发真实交易

任何超出 guardrail 的建议都必须进入人工确认或被 orchestrator 拒绝采纳。

### 5. 明确执行绑定与复用策略

Python 代码仍然会被大量复用，但它们属于 executor / adapter 层，例如：

- ETF 行情与历史数据抓取
- 指标与动量计算
- 回测与交易执行框架

也就是说，**代码实现是 skill 的执行绑定，不是 skill 本体**。

## Capability Scope

本次变更会重写现有 delta specs，并补齐两份缺失的 specs：

- 新增 `portfolio-constructor/spec.md`
- 新增 `decision-explainer/spec.md`

这样可以让 `specs/` 与 `design.md` 中声明的核心 skill 集保持一致。

## Out of Scope

本次不包含以下内容：

- 把 skills 固化成新的 `instock/skills/` 模块树
- 允许 AI 直接控制真实交易
- 为此引入新的重型外部基础设施
- 改写现有全站数据抓取或交易底层协议

## Impact

### 对现有系统的影响

- 现有抓取、指标、回测、交易代码主要作为 executor / adapter 复用
- 新增的复杂度主要集中在 manifest、registry、runtime、approval、audit
- backtest / paper / live 三种模式会拥有统一的 decision envelope，但 side effects 权限不同

### 对文档的影响

- `proposal.md` 不再描述“新增若干 Python 模块”
- `specs/*.md` 不再描述“函数级 API”
- 全部文档统一采用 skill contract、编排边界、权限与审计术语

### 风险与收益

- 风险：初期需要先建立 contract 与 runtime 规范
- 收益：后续 skill 扩展、审计、回测一致性、live 风控边界都会更清晰