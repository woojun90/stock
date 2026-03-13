# ETF轮动策略 - 文档重构总结

## 变更概述

按照 Agent Skill 驱动的架构理念，重构了 OpenSpec 文档结构，确保：
1. **Skill 定义可直接被 AI 消费** - manifest.yaml 格式
2. **共享数据结构统一** - schemas/ 目录
3. **验收规格与实现分离** - spec.md 面向开发者验收

---

## 新目录结构

```
openspec/changes/etf-rotation-agent-strategy/
│
├── proposal.md          # 变更提案（保持不变）
├── design.md            # 架构设计（已精简，指向manifest和schemas）
├── tasks.md             # 实现任务（已重构）
├── analysis.md          # 原方案分析（保留供参考）
│
├── schemas/             # 【新增】共享数据结构定义
│   ├── market-snapshot.yaml       # 市场快照
│   ├── candidate-ranking.yaml     # 候选排名
│   ├── risk-state.yaml            # 风险状态
│   ├── target-portfolio.yaml      # 目标组合
│   ├── decision-envelope.yaml     # 决策信封
│   └── skill-result-envelope.yaml # Skill执行结果
│
└── skills/              # 【重构】每个skill一个目录
    │
    ├── market-data-snapshot/
    │   ├── manifest.yaml          # Skill定义（AI可消费）
    │   └── spec.md                # 验收规格
    │
    ├── momentum-ranking/
    │   ├── manifest.yaml
    │   └── spec.md
    │
    ├── risk-guard/
    │   ├── manifest.yaml
    │   └── spec.md
    │
    ├── market-context-interpreter/
    │   ├── manifest.yaml
    │   └── spec.md
    │
    ├── portfolio-constructor/
    │   ├── manifest.yaml
    │   └── spec.md
    │
    ├── rebalance-executor/
    │   ├── manifest.yaml
    │   └── spec.md
    │
    ├── decision-explainer/
    │   ├── manifest.yaml
    │   └── spec.md
    │
    └── rotation-orchestrator/
        ├── manifest.yaml
        └── spec.md
```

---

## 核心改进

### 1. Skill Manifest 格式

每个 `manifest.yaml` 包含：
- `skill_id`, `purpose`, `mode` - 能力标识
- `input_schema`, `output_schema` - 输入输出（引用 schemas/）
- `preconditions` - 执行前置条件
- `side_effects`, `permissions` - 副作用与权限
- `fallback_policy` - 降级策略
- `executor_binding` - Python/AI 执行器绑定
- `ai_guardrails` - AI 边界约束
- `mode_behavior` - backtest/paper/live 行为差异

### 2. 共享 Schema 定义

`schemas/` 目录下统一定义所有数据结构：
- 确保各 skill 输入输出一致
- 支持引用 ($ref)
- 可用于运行时验证

### 3. AI 边界强制执行

在 `ai_guardrails` 中声明：
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

### 4. 运行模式矩阵

在 `mode_behavior` 中定义三种模式差异：
- `backtest`: AI 禁用，模拟执行
- `paper`: AI 启用，模拟执行
- `live`: AI 启用，真实执行（需审批）

---

## 与原方案对比

| 方面 | 原方案 | 新方案 |
|-----|-------|--------|
| Skill 定义 | 分散在 spec.md 的 Requirement 中 | 集中在 manifest.yaml |
| 数据结构 | 文字描述 | YAML Schema 定义 |
| AI 边界 | 口头约束"不得..." | manifest 中声明，运行时强制 |
| Executor 绑定 | 无 | executor_binding 字段 |
| 模式差异 | 无 | mode_behavior 字段 |

---

## 下一步

1. **Review**: 确认 manifest.yaml 格式是否符合预期
2. **Implement**: 按 tasks.md 阶段实现
3. **Validate**: 实现 manifest schema 验证脚本
4. **Test**: 验证 skills pipeline 运行

---

*重构时间: 2026-03-13*
