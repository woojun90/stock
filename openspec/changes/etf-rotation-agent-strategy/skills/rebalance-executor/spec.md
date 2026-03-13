# Rebalance Executor Skill Specification

> Canonical skill_id: `rebalance-executor`

## ADDED Requirements

### Requirement: 调仓执行必须区分提案与执行

系统 SHALL 将调仓处理定义为 `rebalance-executor` skill，并区分 `trade_proposal` 与 `trade_execute` 两种副作用级别。

#### Scenario: 生成调仓提案

- WHEN orchestrator 提供 `target_portfolio`、当前持仓与交易窗口
- THEN `rebalance-executor` SHALL 生成结构化 `trade_proposal`
- AND proposal SHALL 包含目标变更、订单草案、滑点约束、执行前检查结果与审计摘要

### Requirement: live execute 必须经过 approval gate

`rebalance-executor` 在 live 模式下 SHALL NOT 在未获批准时直接发送真实订单。

#### Scenario: live 模式等待审批

- WHEN 运行模式为 `live`
- AND `approval_state` 不是 approved
- THEN skill SHALL 停止在 proposal 阶段
- AND 输出 SHALL 明确为 `awaiting_approval`
- AND execution trace SHALL 记录被 gate 拦截的状态

### Requirement: 不同模式下执行语义必须清晰

`rebalance-executor` SHALL 在 backtest、paper、live 三种模式下有不同且可审计的执行语义。

#### Scenario: backtest 模拟执行

- WHEN 运行模式为 `backtest`
- THEN skill SHALL 只产生模拟成交结果与成本估计
- AND SHALL NOT 触发真实交易 side effect

#### Scenario: paper 模式仅生成计划

- WHEN 运行模式为 `paper`
- THEN skill MAY 生成 proposal 与模拟回执
- BUT SHALL NOT 调用真实券商执行接口