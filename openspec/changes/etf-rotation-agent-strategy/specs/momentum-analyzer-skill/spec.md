# Momentum Analyzer Skill Specification

> Canonical skill_id: `momentum-ranking`

## ADDED Requirements

### Requirement: 动量分析必须是可复现的确定性 skill

系统 SHALL 将 ETF 动量排序定义为确定性 skill `momentum-ranking`，并保证在相同输入窗口下可复现。

#### Scenario: 基于 snapshot 计算排名

- WHEN `market_snapshot` 满足历史窗口与数据完整性要求
- THEN `momentum-ranking` SHALL 基于策略 policy 计算收益率、标准化得分、趋势过滤结果与候选排名
- AND 输出 SHALL 不依赖 AI 推理或人工自由文本输入

### Requirement: 动量 skill 必须输出排名与排除理由

`momentum-ranking` SHALL 输出结构化候选集，而不是只输出最终买入代码列表。

#### Scenario: 输出结构化 ranking

- WHEN skill 完成计算
- THEN 输出 SHALL 包含 `ranked_candidates`、`score_breakdown`、`excluded_candidates`、`exclusion_reasons` 与 `reproducibility_metadata`
- AND 每个 ETF 的排除原因 SHALL 可被审计

### Requirement: 数据不足不得被 AI 自动补写为确定性结果

`momentum-ranking` SHALL 在数据不足时返回可解释的降级结果，并禁止 AI skill 直接替代其确定性输出。

#### Scenario: 历史窗口不足

- WHEN 某 ETF 缺少满足动量窗口的历史数据
- THEN 该 ETF SHALL 被标记为 excluded
- AND exclusion reason SHALL 明确为 `insufficient_history`
- AND `market-context-interpreter` 或其他 AI skill SHALL NOT 直接将其改写回 ranked list