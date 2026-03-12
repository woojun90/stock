# Decision Explainer Specification

> Canonical skill_id: `decision-explainer`

## ADDED Requirements

### Requirement: 决策解释必须是独立 AI skill

系统 SHALL 提供 AI-assisted skill `decision-explainer`，用于把结构化决策输出转换为人类可读的解释与复核材料。

#### Scenario: 生成人类可读摘要

- WHEN orchestrator 已形成 `decision envelope`
- THEN `decision-explainer` SHALL 生成包含调仓理由、风险说明、关键约束与待审批事项的解释文本
- AND 输出 SHALL 保留对结构化字段的引用关系

### Requirement: 解释 skill 不得改写最终决策

`decision-explainer` SHALL 只解释已有决策，不得反向修改 `target_portfolio`、`trade_proposal` 或审批状态。

#### Scenario: 输出解释但不改写结果

- WHEN `decision-explainer` 被调用
- THEN 它 MAY 增加解释、标签与复核建议
- BUT SHALL NOT 修改任何确定性结果字段

### Requirement: 解释必须覆盖风险与审批信息

`decision-explainer` SHALL 明确呈现本轮决策的风险与审批边界，避免把 AI 文本误认为执行授权。

#### Scenario: live 提案进入人工确认

- WHEN 当前 cycle 产生 live `trade_proposal`
- THEN explanation SHALL 明确标注 `awaiting approval`
- AND SHALL 汇总触发的风险约束、AI advisory 与人工待确认事项