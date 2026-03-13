# ETF轮动策略 - 实现任务清单

> 目标：把 ETF 轮动框架实现为 **agent skills 驱动**的策略系统。

## 阶段 1: Schema 与 Skill Manifest 基础

- [x] 1.1 创建 `schemas/` 目录及共享数据结构定义
  - [x] `market-snapshot.yaml`
  - [x] `candidate-ranking.yaml`
  - [x] `risk-state.yaml`
  - [x] `target-portfolio.yaml`
  - [x] `decision-envelope.yaml`
  - [x] `skill-result-envelope.yaml`

- [x] 1.2 创建各 skill 的 `manifest.yaml`
  - [x] `skills/market-data-snapshot/manifest.yaml`
  - [x] `skills/momentum-ranking/manifest.yaml`
  - [x] `skills/risk-guard/manifest.yaml`
  - [x] `skills/market-context-interpreter/manifest.yaml`
  - [x] `skills/portfolio-constructor/manifest.yaml`
  - [x] `skills/rebalance-executor/manifest.yaml`
  - [x] `skills/decision-explainer/manifest.yaml`
  - [x] `skills/rotation-orchestrator/manifest.yaml`

- [x] 1.3 迁移现有 specs 到新目录结构
  - [x] 各 skill 目录下添加 `spec.md`

- [ ] 1.4 实现 manifest schema 验证
  - [ ] 定义 manifest.yaml 的 JSON Schema
  - [ ] 实现验证脚本

---

## 阶段 2: Skill Registry 与 Runtime 核心

- [ ] 2.1 实现 Skill Registry
  - [ ] manifest 加载器 (从 YAML 加载到内存)
  - [ ] 按 skill_id / trigger / mode / permission 查询
  - [ ] schema 引用解析 (加载 schemas/ 下的定义)

- [ ] 2.2 实现 StrategyContext
  - [ ] 定义 context 字段结构
  - [ ] 实现 context 创建与冻结
  - [ ] 实现字段写入权限控制

- [ ] 2.3 实现 WorkingMemory
  - [ ] 定义临时推理区结构
  - [ ] advisory_annotations 支持
  - [ ] human_review_queue 支持

- [ ] 2.4 实现 SkillResultEnvelope
  - [ ] 统一返回结构
  - [ ] 状态码定义 (success/degraded/failed/skipped/no_op)
  - [ ] 错误与警告结构

- [ ] 2.5 实现 Execution Trace
  - [ ] 记录每轮调用的 skills 列表
  - [ ] 记录输入/输出摘要
  - [ ] 记录耗时与状态

---

## 阶段 3: Orchestrator 核心

- [ ] 3.1 实现 Cycle 生命周期
  - [ ] cycle_id 生成
  - [ ] context 初始化
  - [ ] mode 判断

- [ ] 3.2 实现 Skill Pipeline 编排
  - [ ] 按 pipeline 定义顺序调度 skills
  - [ ] 处理 required vs optional skill
  - [ ] 处理 fail-fast vs degrade-on-fail

- [ ] 3.3 实现 Preconditions 检查
  - [ ] 解析 manifest 中的 preconditions
  - [ ] 运行时求值

- [ ] 3.4 实现 Conflict Resolution
  - [ ] 按优先级合并输出
  - [ ] 记录冲突决策

- [ ] 3.5 实现 Decision Envelope 聚合
  - [ ] 合并各 skill 输出
  - [ ] 生成 idempotency_key

---

## 阶段 4: 确定性 Executors

- [ ] 4.1 market-data-snapshot executor
  - [ ] 复用 `fund_etf_em.py`
  - [ ] 实现 snapshot 生成
  - [ ] 实现 cache 降级

- [ ] 4.2 momentum-ranking executor
  - [ ] 复用指标计算
  - [ ] 实现 Z-Score 标准化
  - [ ] 实现趋势过滤

- [ ] 4.3 risk-guard executor
  - [ ] 实现止损检查
  - [ ] 实现回撤检查
  - [ ] 实现冷却期检查
  - [ ] 实现仓位限制

- [ ] 4.4 portfolio-constructor executor
  - [ ] 实现权重分配
  - [ ] 应用约束
  - [ ] 生成 target_portfolio

- [ ] 4.5 rebalance-executor executor
  - [ ] 实现 trade_proposal 生成
  - [ ] 实现 backtest 模拟成交
  - [ ] 实现 paper 模拟回执
  - [ ] 实现与 easytrader 集成

---

## 阶段 5: AI Skill Executors

- [ ] 5.1 AI Executor 基础框架
  - [ ] Claude API 集成
  - [ ] 输入裁剪
  - [ ] 输出结构化
  - [ ] 超时处理

- [ ] 5.2 market-context-interpreter executor
  - [ ] 新闻/政策数据获取
  - [ ] AI 推理 prompt
  - [ ] 输出结构化

- [ ] 5.3 decision-explainer executor
  - [ ] 读取 decision_envelope
  - [ ] 生成人类可读解释
  - [ ] 降级为模板化解释

- [ ] 5.4 AI Boundary Guardrails
  - [ ] 运行时检查 writable_fields
  - [ ] 运行时检查 forbidden_fields
  - [ ] 越界时拒绝或警告

---

## 阶段 6: Approval Gate 与审计

- [ ] 6.1 Approval Gate
  - [ ] 实现审批状态机
  - [ ] 实现 approval_token 生成
  - [ ] 实现过期检查
  - [ ] 实现幂等键校验

- [ ] 6.2 人工确认
  - [ ] 实现确认界面/通知
  - [ ] 实现单笔 vs 批量审批

- [ ] 6.3 审计日志
  - [ ] execution_trace 持久化
  - [ ] decision_log 持久化
  - [ ] 查询接口

---

## 阶段 7: 运行模式

- [ ] 7.1 Backtest 模式
  - [ ] 历史数据回放
  - [ ] AI skill 禁用/no-op
  - [ ] 模拟成交

- [ ] 7.2 Paper 模式
  - [ ] 实时数据 + 模拟执行
  - [ ] AI skill 启用
  - [ ] 不触达券商

- [ ] 7.3 Live 模式
  - [ ] 审批闸门强制
  - [ ] 真实执行
  - [ ] 异常告警

---

## 阶段 8: 测试与验收

- [ ] 8.1 Schema 验证测试
- [ ] 8.2 Manifest 加载测试
- [ ] 8.3 确定性 skill 回测一致性测试
- [ ] 8.4 AI boundary guardrails 测试
- [ ] 8.5 Approval gate 测试
- [ ] 8.6 Pipeline 编排集成测试

---

## 里程碑

| 里程碑 | 完成条件 | 解锁能力 |
|-------|---------|---------|
| **M1: Schema + Manifest** | 阶段1完成 | 文档结构就绪 |
| **M2: Registry + Context** | 阶段2完成 | 可加载和调度skill |
| **M3: Orchestrator** | 阶段3完成 | 可运行pipeline |
| **M4: Deterministic Skills** | 阶段4完成 | 可回测确定性策略 |
| **M5: AI Skills** | 阶段5完成 | AI advisory可用 |
| **M6: Approval + Live** | 阶段6-7完成 | 可实盘运行 |

**最小可运行里程碑**: M4 (完成确定性skills + orchestrator，可运行backtest)

---

## 设计红线

1. **不得把 skill 本体退化为 Python 模块列表** - skill 定义在 manifest.yaml
2. **Python 代码是 executor/adapter，不是 skill 本体** - 通过 executor_binding 绑定
3. **AI 不得绕过 risk-guard** - ai_guardrails 强制执行
4. **live 执行必须经过 approval gate** - 不得偷偷调用交易接口
