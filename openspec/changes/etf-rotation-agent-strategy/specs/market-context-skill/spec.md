# Market Context Skill Specification

> Canonical skill_id: `market-context-interpreter`

## ADDED Requirements

### Requirement: 市场环境解读必须是 advisory-only AI skill

系统 SHALL 将市场环境解读定义为 AI-assisted skill `market-context-interpreter`，其权限级别 SHALL 为 `advisory`。

#### Scenario: 生成市场环境说明

- WHEN orchestrator 在开盘前或调仓前请求市场背景解读
- THEN `market-context-interpreter` SHALL 输出结构化的 `market_context`
- AND 输出 MAY 包含 regime judgement、政策风险提示、异常事件摘要与建议性约束
- AND 输出 SHALL 带有 reasoning summary 与 source references

### Requirement: AI skill 不得直接改写确定性交易主路径

`market-context-interpreter` SHALL NOT 直接修改 `market_snapshot`、`candidate_rankings`、`risk_state` 或 `target_portfolio` 的确定性结果。

#### Scenario: AI 提出偏离建议

- WHEN AI 输出建议降低某类 ETF 权重或暂停调仓
- THEN 建议 SHALL 以 advisory annotation 的形式附着到 decision context
- AND orchestrator SHALL 依据 policy 与 approval rules 决定是否采纳
- AND AI 输出 SHALL NOT 直接覆盖 ranking 或 risk hard constraints

### Requirement: 回测模式下 AI skill 必须可禁用或回放

AI skill SHALL 在 backtest 中具备可控行为，以保证确定性路径可重复。

#### Scenario: Backtest 默认禁用 AI

- WHEN orchestrator 运行于 backtest mode
- THEN `market-context-interpreter` SHALL 默认返回 no-op、录制回放结果或显式禁用状态
- AND backtest 结果 SHALL 不因在线 AI 响应而漂移