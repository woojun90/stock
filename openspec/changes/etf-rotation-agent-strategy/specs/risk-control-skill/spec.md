# Risk Control Skill Specification

> Canonical skill_id: `risk-guard`

## ADDED Requirements

### Requirement: 风控必须是确定性主路径的一部分

系统 SHALL 将 ETF 轮动风控建模为确定性 skill `risk-guard`，并在 portfolio 构建或交易提案之前执行。

#### Scenario: 在调仓前执行风险约束

- WHEN orchestrator 已获得候选排名、当前持仓与账户状态
- THEN `risk-guard` SHALL 评估止损、回撤、波动率、冷却期与仓位上限
- AND 输出 SHALL 形成统一的 `risk_state`

### Requirement: 风控必须能输出硬约束与风险事件

`risk-guard` SHALL 产出可执行的硬约束，而不是只给出模糊风险评级。

#### Scenario: 生成禁止买入与减仓提案

- WHEN 某 ETF 触发冷却期、波动率阈值或止损规则
- THEN 输出 SHALL 包含 `prohibited_buys`、`required_reductions`、`forced_exits` 与 `risk_events`
- AND 每项约束 SHALL 附带触发规则与阈值来源

### Requirement: 风控优先级必须高于 AI advisory

`risk-guard` 的硬约束 SHALL 优先于 AI skill 的建议性输出。

#### Scenario: AI 建议与风控冲突

- WHEN `market-context-interpreter` 建议保留或加仓某 ETF
- AND `risk-guard` 已要求该 ETF 禁买或退出
- THEN orchestrator SHALL 以 `risk-guard` 为准
- AND execution trace SHALL 记录该 advisory 被覆盖的原因