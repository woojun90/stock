# ETF轮动策略 - 变更提案

## Why

### 问题陈述

当前 InStock 系统缺少 ETF 轮动策略框架。传统实现方式存在以下问题：

1. **策略规则硬编码** - 修改策略需要改代码，灵活性差
2. **AI能力与交易边界模糊** - AI可能直接影响交易决策，风险不可控
3. **回测与实盘行为不一致** - AI参与时结果不可复现
4. **缺乏审计追溯** - 难以理解决策是如何形成的

### 解决方向

构建一个 **由 Agent Skills 能力契约驱动** 的 ETF 轮动框架：
- Skill 是声明式能力定义，不是 Python 模块
- Orchestrator 基于 Registry 编排，不依赖具体实现
- 确定性 skill 保证回测一致，AI skill 只做 advisory
- Decision Envelope 提供完整审计追溯

---

## What Changes

### 1. 引入 Skill Manifest 作为一等抽象

每个 skill 通过 `manifest.yaml` 定义：
- 能力标识与用途 (`skill_id`, `purpose`)
- 运行模式 (`mode`: deterministic / ai_assisted)
- 输入输出 Schema (引用 `schemas/`)
- 前置条件与副作用 (`preconditions`, `side_effects`)
- 权限级别 (`permissions`)
- AI 边界约束 (`ai_guardrails`)
- 执行器绑定 (`executor_binding`)

### 2. 建立共享 Schema 体系

`schemas/` 目录定义统一数据结构：
- `market-snapshot.yaml` - 市场快照
- `candidate-ranking.yaml` - 候选排名
- `risk-state.yaml` - 风险状态
- `target-portfolio.yaml` - 目标组合
- `decision-envelope.yaml` - 决策信封

### 3. 实现 Skills-first 编排架构

```
Policy Layer (YAML配置)
    ↓
Skill Definition Layer (manifest.yaml)
    ↓
Skill Runtime Layer (Registry, Orchestrator, Context, Approval)
    ↓
Executor / Adapter Layer (Python, AI)
```

### 4. 定义首批 ETF 轮动 Skills

| Skill | Mode | 用途 |
|-------|------|------|
| `market-data-snapshot` | deterministic | 生成市场快照 |
| `momentum-ranking` | deterministic | 动量排名计算 |
| `risk-guard` | deterministic | 风险约束评估 |
| `market-context-interpreter` | ai_assisted | 市场环境解读 (advisory) |
| `portfolio-constructor` | deterministic | 目标组合构建 |
| `rebalance-executor` | deterministic | 调仓执行 |
| `decision-explainer` | ai_assisted | 决策解释 (advisory) |
| `rotation-orchestrator` | deterministic | 编排中心 |

### 5. 建立 AI 边界强制执行机制

AI skill 在 manifest 中声明 `ai_guardrails`：
- `writable_fields`: 允许写入的字段
- `forbidden_fields`: 禁止写入的字段
- Orchestrator 运行时强制执行

### 6. 实现多模式运行

| Mode | AI Skills | Approval Gate | Execution |
|------|-----------|---------------|-----------|
| backtest | 禁用/no-op | 绕过 | 模拟成交 |
| paper | 启用(advisory) | 模拟 | 模拟回执 |
| live | 启用(advisory) | 强制 | 审批后真实执行 |

---

## Capability Scope

### 核心能力

1. **etf-data-skill** - ETF 数据获取与快照生成
2. **momentum-analyzer-skill** - 动量分析与排名
3. **risk-control-skill** - 风险控制与约束
4. **market-context-skill** - 市场环境解读
5. **portfolio-constructor** - 组合构建
6. **rebalance-executor-skill** - 调仓执行
7. **decision-explainer** - 决策解释
8. **rotation-orchestrator** - 策略编排

### 配置能力

- `config/policy.yaml` - 策略配置

---

## Out of Scope

1. 不把 skill 等同于独立 Python 模块目录
2. 不允许 AI 直接绕过风控或直接下单
3. 不引入新的重型外部基础设施
4. 不改造现有数据源与交易底层协议

---

## Impact

### 对现有系统的影响

- 现有抓取、指标、回测、交易代码作为 executor/adapter 复用
- 新增复杂度集中在 manifest、registry、runtime、approval、audit
- backtest / paper / live 三种模式统一 decision envelope

### 风险与收益

| 风险 | 缓解措施 |
|-----|---------|
| Skill 定义过于抽象 | manifest.yaml 必须可验证 |
| AI 建议侵入确定性路径 | ai_guardrails 运行时强制 |
| Runtime 复杂度上升 | 分阶段实现，复用现有基础设施 |

---

## 文档结构

```
openspec/changes/etf-rotation-agent-strategy/
├── proposal.md              # 本文档
├── design.md                # 架构设计
├── tasks.md                 # 实现任务
├── ARCHITECTURE.md          # 架构概览图
│
├── schemas/                 # 共享数据结构
│   ├── market-snapshot.yaml
│   ├── candidate-ranking.yaml
│   ├── risk-state.yaml
│   ├── target-portfolio.yaml
│   ├── decision-envelope.yaml
│   ├── skill-result-envelope.yaml
│   └── skill-manifest.yaml
│
├── config/                  # 策略配置
│   └── policy.yaml
│
└── skills/                  # Skill 定义
    ├── market-data-snapshot/
    ├── momentum-ranking/
    ├── risk-guard/
    ├── market-context-interpreter/
    ├── portfolio-constructor/
    ├── rebalance-executor/
    ├── decision-explainer/
    └── rotation-orchestrator/
```
