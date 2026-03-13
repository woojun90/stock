# Rotation Orchestrator Specification

> Capability id: `rotation-orchestrator`

## ADDED Requirements

### Requirement: orchestrator 必须面向 registry 与 context 编排

系统 SHALL 通过 `rotation-orchestrator` 基于 skill registry、strategy policy 与共享 context 驱动整条 ETF 轮动链路，而不是直接硬编码模块调用。

#### Scenario: 加载技能并建立上下文

- WHEN 新的一轮调仓 cycle 被触发
- THEN orchestrator SHALL 加载可用 skill manifests、policy 与 mode 配置
- AND SHALL 初始化 `StrategyContext`、`WorkingMemory` 与 `execution_trace`

### Requirement: orchestrator 必须生成统一的 decision envelope

`rotation-orchestrator` SHALL 聚合各个 skills 的结果，形成可审计的 `decision envelope`。

#### Scenario: 构建主路径决策

- WHEN 周度调仓主流程运行
- THEN orchestrator SHALL 按 `market-data-snapshot → momentum-ranking → risk-guard → optional market-context-interpreter → portfolio-constructor → optional decision-explainer → rebalance-executor` 的阶段顺序调度 skills
- AND SHALL 将关键输入摘要、输出摘要、fallback、审批状态与最终建议写入 `decision envelope`

### Requirement: orchestrator 必须执行冲突优先级规则

系统 SHALL 对冲突输出采用统一优先级：`risk-guard > policy guardrail > AI advisory > ranking result`。

#### Scenario: 候选排名与风控冲突

- WHEN ranking 结果建议买入某 ETF
- AND risk guard 或 policy guardrail 禁止买入
- THEN orchestrator SHALL 拒绝将该 ETF 纳入最终目标组合
- AND decision envelope SHALL 记录冲突决策原因

### Requirement: orchestrator 必须支持 mode-aware 行为切换

`rotation-orchestrator` SHALL 根据 backtest、paper、live 模式切换 AI、审批和执行行为。

#### Scenario: live 模式触发审批

- WHEN cycle 进入真实交易阶段
- THEN orchestrator SHALL 检查 approval gate 与 permission requirements
- AND 在未满足条件时停止于 proposal 或人工确认阶段