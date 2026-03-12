# ETF轮动策略 - 实现任务清单

> 目标：把 ETF 轮动框架实现为 **agent skills 驱动**的策略系统。以下任务以“能力定义、运行时编排、执行适配、审计与验证”为主线，不再把 skill 简化为若干 Python 模块文件。

## 1. 建立 Skill 能力模型

- [ ] 1.1 定义 skill manifest 的统一 schema（id、purpose、mode、preconditions、input/output、side effects、permissions、executor binding、observability）
- [ ] 1.2 定义 ETF 轮动共享上下文 schema（market snapshot、candidate rankings、risk state、target portfolio、approval state、execution trace）
- [ ] 1.3 定义 skill 执行结果的统一 envelope 与错误模型
- [ ] 1.4 定义 side effect 与 permission taxonomy（read_only / advisory / state_write / trade_proposal / trade_execute）
- [ ] 1.5 定义 backtest / live / paper 三种运行模式下的 skill 可用性矩阵

## 2. 定义首批 ETF 轮动 Skills

- [ ] 2.1 定义 `market-data-snapshot` skill manifest
- [ ] 2.2 定义 `momentum-ranking` skill manifest
- [ ] 2.3 定义 `risk-guard` skill manifest
- [ ] 2.4 定义 `market-context-interpreter` skill manifest
- [ ] 2.5 定义 `portfolio-constructor` skill manifest
- [ ] 2.6 定义 `rebalance-executor` skill manifest
- [ ] 2.7 定义 `decision-explainer` skill manifest
- [ ] 2.8 为每个 skill 明确 preconditions、fallback policy 与 observability 要求

## 3. 实现 Skill Registry 与最小 Runtime

- [ ] 3.1 实现 skill manifest 加载与校验
- [ ] 3.2 实现 registry 查询能力（按 skill_id、trigger、mode、permission 查询）
- [ ] 3.3 实现 `StrategyContext` 与 `WorkingMemory`
- [ ] 3.4 实现 skill 调用标准接口，统一输入输出 envelope
- [ ] 3.5 实现 execution trace 记录，能追踪每轮调仓调用过哪些 skills
- [ ] 3.6 实现 skill 失败处理策略（fail-fast / degrade / retry / human-review）

## 4. 接入确定性 Executors / Adapters

- [ ] 4.1 为 ETF 行情、历史 K 线、资产池加载建立 market data adapter，复用现有数据抓取能力
- [ ] 4.2 为动量、均线趋势、排名建立 deterministic executor，复用现有指标与回测能力
- [ ] 4.3 为回撤、波动率、冷却期、仓位限制建立 risk executor
- [ ] 4.4 为目标权重分配建立 portfolio constructor executor
- [ ] 4.5 为模拟交易与真实交易建立统一 trade executor 接口
- [ ] 4.6 明确 executor binding 如何从 manifest 映射到 Python 实现

## 5. 实现 Orchestrator 编排主链路

- [ ] 5.1 实现周度调仓主流程：snapshot → ranking → risk → optional context → portfolio → explain → execute
- [ ] 5.2 实现日内风险监控流程，只调度与风险相关的 skills
- [ ] 5.3 实现基于 preconditions 的 skill 可用性判断
- [ ] 5.4 实现 `decision envelope` 聚合逻辑
- [ ] 5.5 实现冲突处理优先级：risk guard > policy guardrail > AI advisory > ranking result
- [ ] 5.6 实现手动触发与定时触发的统一调度入口

## 6. 接入 AI 辅助 Skills 与 Guardrails

- [ ] 6.1 实现 `market-context-interpreter` 的 advisory-only 输出边界
- [ ] 6.2 实现 `decision-explainer`，输出人类可读的调仓理由与风险说明
- [ ] 6.3 实现 AI skill 的输入裁剪、输出结构化和日志记录
- [ ] 6.4 实现 guardrail，禁止 AI 直接绕过 risk guard 或直接执行交易
- [ ] 6.5 实现 AI 输出超出阈值时转人工确认的机制

## 7. 审批、权限与审计

- [ ] 7.1 实现 approval gate，对 `trade_proposal` / `trade_execute` 类 skill 进行拦截与审批
- [ ] 7.2 实现 live 模式下的人工确认策略（整轮确认或单笔确认）
- [ ] 7.3 记录每次 skill 执行的输入摘要、输出摘要、耗时、错误与审批状态
- [ ] 7.4 记录本轮调仓由哪些 skill 输出共同构成最终决策
- [ ] 7.5 提供查询最近 execution trace 与 decision audit 的接口

## 8. 回测与实盘一致性

- [ ] 8.1 实现 deterministic skill 图在 backtest 中的完整回放
- [ ] 8.2 实现 AI skill 在 backtest 中的禁用 / no-op / 回放策略
- [ ] 8.3 验证同一历史窗口下 deterministic 决策可重复
- [ ] 8.4 实现 paper 模式，允许生成 proposal 但不执行真实交易
- [ ] 8.5 统一 backtest/live 的 decision envelope 格式

## 9. 策略配置与 Skill Policy

- [ ] 9.1 设计策略 policy 配置（资产池、调仓频率、信号参数、风险阈值、审批规则）
- [ ] 9.2 设计 skill manifests 的配置目录与加载方式
- [ ] 9.3 实现 policy 校验与默认值回退
- [ ] 9.4 实现配置变更后的版本记录与 reload 机制
- [ ] 9.5 明确哪些配置影响 deterministic path，哪些只影响 advisory path

## 10. 持久化与可观测性

- [ ] 10.1 明确需要持久化的对象：signal log、risk event、execution trace、approval record、position snapshot
- [ ] 10.2 设计执行指标与监控项（成功率、耗时、审批通过率、AI advisory 覆盖率）
- [ ] 10.3 实现异常告警与关键决策通知
- [ ] 10.4 生成面向用户的调仓摘要与日报输出

## 11. 测试与验收

- [ ] 11.1 为 manifest schema、registry、context、decision envelope 编写单元测试
- [ ] 11.2 为 deterministic skills 编写回测一致性测试
- [ ] 11.3 为 approval gate 与 guardrail 编写安全性测试
- [ ] 11.4 为 orchestrator 主链路编写集成测试
- [ ] 11.5 为 paper/live 切换编写模式测试
- [ ] 11.6 准备一组可复用的示例 skill manifests 与策略样例

## 12. 文档同步

- [ ] 12.1 更新 proposal，使其明确“skill 是能力契约而非 Python 模块”
- [ ] 12.2 更新 `specs/` 下各 capability 文档，使其采用 skills-first 术语和 contract 结构
- [ ] 12.3 补充一份“如何新增一个 skill”的开发说明
- [ ] 12.4 补充一份“如何调试一次调仓 cycle”的运行说明

---

**实施主线**：1 → 2 → 3 → 4 → 5 → 7 → 8 → 11

**最小可运行里程碑**：完成 `market-data-snapshot`、`momentum-ranking`、`risk-guard`、`portfolio-constructor`、`rebalance-executor(simulated)` + 最小 orchestrator

**设计红线**：不得把 skill 本体退化为 `instock/skills/*.py` 的模块列表；任何 Python 实现都应被视为 executor / adapter，而不是 skill 定义本身。