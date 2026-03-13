# ETF轮动策略 - 技术设计文档

## Context

### 背景

本项目目标：构建一个**由 Agent Skills 能力定义驱动**的 ETF 轮动策略框架。

现有 InStock 具备以下基础设施，可作为 skill 的底层执行资源：
- ETF 行情与历史数据获取 (`fund_etf_em.py`)
- 技术指标计算与回测基础能力 (`calculate_indicator.py`)
- 交易执行框架 (`trade/robot/`)
- MySQL 持久化能力

### 文档结构

```
openspec/changes/etf-rotation-agent-strategy/
├── proposal.md          # 变更提案
├── design.md            # 本文档 - 架构决策
├── tasks.md             # 实现任务清单
├── schemas/             # 共享数据结构定义 (YAML Schema)
│   ├── market-snapshot.yaml
│   ├── candidate-ranking.yaml
│   ├── risk-state.yaml
│   ├── target-portfolio.yaml
│   ├── decision-envelope.yaml
│   └── skill-result-envelope.yaml
└── skills/              # 每个skill一个目录
    ├── <skill-id>/
    │   ├── manifest.yaml   # skill定义 (AI可消费)
    │   └── spec.md         # 验收规格 (开发者验收)
```

---

## Goals / Non-Goals

### Goals

1. **Skill 是能力契约，不是 Python 模块**
2. **Orchestrator 依赖 Registry 编排，不依赖具体实现**
3. **核心交易逻辑确定性可回测**
4. **AI skill 只做 advisory，不得绕过风控**

### Non-Goals

1. 不把 skill 等同于独立 Python 模块目录
2. 不让 AI 直接绕过风控或直接下单
3. 不引入新的重型外部基础设施

---

## Architecture

### 分层架构

```
┌─────────────────────────────────────────────────────────────────┐
│ Policy Layer                                                    │
│   ├─ strategy policy (YAML配置)                                 │
│   ├─ ETF universe                                               │
│   └─ guardrails / approval rules                                │
├─────────────────────────────────────────────────────────────────┤
│ Skill Definition Layer                                          │
│   ├─ manifest.yaml (每个skill的定义)                            │
│   ├─ schemas/ (共享数据结构)                                    │
│   └─ spec.md (验收规格)                                         │
├─────────────────────────────────────────────────────────────────┤
│ Skill Runtime Layer                                             │
│   ├─ registry (加载manifest)                                    │
│   ├─ orchestrator (编排执行)                                    │
│   ├─ strategy context (共享上下文)                              │
│   ├─ approval gate (审批闸门)                                   │
│   └─ audit / trace (审计追踪)                                   │
├─────────────────────────────────────────────────────────────────┤
│ Executor / Adapter Layer                                        │
│   ├─ Python executors (绑定到manifest.executor_binding)         │
│   ├─ AI executors (Claude API)                                  │
│   └─ 复用现有 InStock 基础设施                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 核心决策

### Decision 1: Skill Manifest 作为一等抽象

每个 skill 必须有 `manifest.yaml`，定义：
- `skill_id`, `purpose`, `mode`
- `input_schema`, `output_schema` (引用 schemas/)
- `preconditions`, `side_effects`, `permissions`
- `fallback_policy`, `executor_binding`, `ai_guardrails`

详见: `skills/*/manifest.yaml`

### Decision 2: 共享 Schema 确保数据一致

所有 skill 输入/输出必须引用 `schemas/` 下的统一定义：
- `market-snapshot.yaml` - 市场快照
- `candidate-ranking.yaml` - 候选排名
- `risk-state.yaml` - 风险状态
- `target-portfolio.yaml` - 目标组合
- `decision-envelope.yaml` - 决策信封
- `skill-result-envelope.yaml` - Skill执行结果

### Decision 3: 确定性 vs AI Skill 分离

| 类型 | Skills | 特点 |
|-----|--------|------|
| **Deterministic** | market-data-snapshot, momentum-ranking, risk-guard, portfolio-constructor, rebalance-executor | 回测/实盘一致，AI不可覆盖 |
| **AI-assisted** | market-context-interpreter, decision-explainer | advisory-only，不得修改确定性路径 |

### Decision 4: AI Boundary Guardrails

AI skill 在 manifest 中声明：
```yaml
ai_guardrails:
  allow_ai_override: false
  writable_fields:
    - market_advisory
    - human_readable_rationale
  forbidden_fields:
    - candidate_rankings
    - risk_constraints
    - target_portfolio
    - trade_proposal
```

Orchestrator 运行时强制执行这些边界。

### Decision 5: 运行模式矩阵

| Skill | Backtest | Paper | Live |
|-------|----------|-------|------|
| market-data-snapshot | 历史回放 | 当日数据 | 实时数据 |
| momentum-ranking | 启用 | 启用 | 启用 |
| risk-guard | 启用 | 启用 | 启用 |
| market-context-interpreter | **禁用/no-op** | 启用(advisory) | 启用(advisory) |
| portfolio-constructor | 启用 | 启用 | 启用 |
| decision-explainer | 模板化 | 启用 | 启用 |
| rebalance-executor | 模拟成交 | 模拟回执 | **审批后真实执行** |

---

## Skill Pipeline

### 周度调仓流程

```
┌──────────────────────────────────────────────────────────────────────┐
│                     Weekly Rebalancing Pipeline                       │
├──────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  1. market-data-snapshot  ──▶  market_snapshot                       │
│     (required, fail-fast)                                             │
│                                                                       │
│  2. momentum-ranking  ──▶  candidate_rankings                        │
│     (required, fail-fast)                                             │
│                                                                       │
│  3. risk-guard  ──▶  risk_state, constraints                         │
│     (required, fail-fast)                                             │
│                                                                       │
│  4. market-context-interpreter  ──▶  market_advisory (optional)      │
│     (optional, degrade-on-fail)                                       │
│                                                                       │
│  5. portfolio-constructor  ──▶  target_portfolio                     │
│     (required, fail-fast)                                             │
│                                                                       │
│  6. decision-explainer  ──▶  human_readable_rationale (optional)     │
│     (optional, degrade-on-fail)                                       │
│                                                                       │
│  7. rebalance-executor  ──▶  trade_proposal / trade_execute          │
│     (required, gate-on-approval)                                      │
│                                                                       │
│  ───────────────────────────────────────────────────────────────────  │
│  Output: decision_envelope (聚合所有结果)                             │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘
```

### 冲突优先级

```
risk-guard > policy_guardrail > ai_advisory > ranking_result
```

高优先级约束覆盖低优先级建议。

---

## Risks / Trade-offs

| 风险 | 缓解措施 |
|-----|---------|
| Skill定义过于抽象 | manifest.yaml 必须可验证，executor_binding 必须明确 |
| AI建议侵入确定性路径 | ai_guardrails 在运行时强制执行 |
| Runtime复杂度上升 | 第一阶段实现最小runtime，复用现有基础设施 |
| Skill间数据口径不一致 | 统一 schemas/ 定义，registry 加 schema validation |

---

## Open Questions

1. `market-context-interpreter` 的外部数据来源范围？
2. 是否支持"整轮审批 + 单笔例外审批"混合模型？
3. skill manifest 放在策略配置目录还是单独目录？

---

> **参考**: 
> - `schemas/` - 共享数据结构定义
> - `skills/*/manifest.yaml` - 各skill详细定义
> - `skills/*/spec.md` - 各skill验收规格
