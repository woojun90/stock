# Portfolio Constructor Specification

> Canonical skill_id: `portfolio-constructor`

## ADDED Requirements

### Requirement: 目标组合构建必须是独立 skill

系统 SHALL 将目标权重分配建模为确定性 skill `portfolio-constructor`，而不是隐含在执行器内部。

#### Scenario: 基于排名与风险约束构建组合

- WHEN orchestrator 已获得 `candidate_rankings` 与 `risk_state`
- THEN `portfolio-constructor` SHALL 生成 `target_portfolio`
- AND 输出 SHALL 包含持仓名单、目标权重、现金比例与约束应用摘要

### Requirement: 组合构建必须保留可审计的分配理由

`portfolio-constructor` SHALL 说明目标权重如何由 policy、ranking 和 risk constraints 共同决定。

#### Scenario: 输出权重分配依据

- WHEN 某 ETF 被纳入目标组合
- THEN 输出 SHALL 记录该 ETF 的入选依据、权重上限来源与被应用的约束
- AND WHEN 某高排名 ETF 未被纳入时
- THEN 输出 SHALL 记录未纳入原因

### Requirement: 组合构建不得直接产生交易 side effect

`portfolio-constructor` SHALL 只负责形成目标组合，不得直接下单或绕过审批流程。

#### Scenario: 交给执行 skill 处理交易

- WHEN `target_portfolio` 已形成
- THEN orchestrator SHALL 将其交由 `rebalance-executor` 处理
- AND `portfolio-constructor` SHALL NOT 直接调用交易接口