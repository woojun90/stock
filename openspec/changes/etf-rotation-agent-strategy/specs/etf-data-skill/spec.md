# ETF Data Skill Specification

> Canonical skill_id: `market-data-snapshot`

## ADDED Requirements

### Requirement: ETF 数据获取必须被建模为 skill contract

系统 SHALL 将 ETF 行情与历史数据获取定义为确定性 skill `market-data-snapshot`，而不是暴露为若干直接供 orchestrator 调用的模块函数。

#### Scenario: Registry 注册数据 skill

- WHEN orchestrator 加载 ETF 轮动 skill manifests
- THEN registry SHALL 注册 `market-data-snapshot`
- AND 该 skill SHALL 声明 `mode=deterministic`
- AND 该 skill SHALL 声明 `permissions=read_only`
- AND 该 skill SHALL 包含输入/输出 schema、fallback policy、executor binding 与 observability 字段

### Requirement: 数据 skill 必须输出统一的 market snapshot

`market-data-snapshot` SHALL 输出统一的 `market_snapshot` 结构，供后续 ranking、risk、portfolio skills 复用。

#### Scenario: 生成结构化 snapshot

- WHEN orchestrator 请求指定 ETF universe 的行情快照
- THEN skill SHALL 返回 `market_snapshot`
- AND 输出 SHALL 包含 `as_of`、`universe_version`、`price_series`、`liquidity_metrics`、`premium_discount`、`data_quality` 与 `source_refs`
- AND 输出 SHALL 标记 freshness 状态，以便下游 skill 判断是否可继续运行

### Requirement: 数据缺失时必须按 fallback policy 降级

`market-data-snapshot` SHALL 在实时数据缺失、延迟或部分 ETF 获取失败时遵循声明式 fallback policy，而不是隐式吞掉异常。

#### Scenario: 使用缓存数据降级

- WHEN 实时抓取失败但存在可接受时间窗内的缓存数据
- THEN skill SHALL 返回带 `freshness_status=stale` 的 snapshot
- AND execution trace SHALL 记录降级原因
- AND 下游 orchestrator SHALL 能够基于 freshness policy 决定继续、重试或转人工

#### Scenario: 无法形成最小可用快照

- WHEN 可用 ETF 数量低于策略要求的最小覆盖率
- THEN skill SHALL 返回结构化失败结果
- AND orchestrator SHALL 中止本轮确定性决策主路径