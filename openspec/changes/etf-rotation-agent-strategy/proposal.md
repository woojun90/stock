# ETF轮动策略 - AI Agent驱动方案

## Why

传统量化策略存在两大痛点：规则硬编码导致灵活性不足，以及面对政策市、黑天鹅等非结构化场景时缺乏适应性。当前项目已有完整的ETF数据获取、指标计算和交易执行基础设施，但缺乏一个能够动态解读规则、主动识别异常、输出决策理由的智能策略框架。

通过引入AI Agent驱动的方式，将策略逻辑从硬编码转变为可配置的规则+AI推理模式，让策略能够在保持核心动量轮动逻辑的同时，具备应对市场异常情况的能力。

## What Changes

### 新增模块

- **Agent Skills 模块** (`instock/skills/`): 独立的功能性技能模块，每个Skill封装特定能力
  - `etf_data.py` - ETF数据获取能力
  - `momentum.py` - 动量分析与信号生成能力
  - `risk_control.py` - 风险监控与预警能力
  - `market_context.py` - 市场环境感知能力
  - `rebalance.py` - 调仓执行能力
  - `notification.py` - 通知与日志能力

- **规则配置系统** (`config/`): YAML格式的策略规则配置
  - `etf_rotation_rules.yaml` - 核心策略规则
  - `etf_pool.yaml` - ETF资产池定义
  - `risk_thresholds.yaml` - 风控阈值配置

- **轮动策略核心** (`instock/core/strategy/etf_rotation.py`): 策略编排逻辑

- **轮动回测引擎** (`instock/core/backtest/rotation_backtest.py`): 专门针对轮动策略的回测框架

- **交易策略入口** (`instock/trade/strategies/etf_rotation_strategy.py`): 继承StrategyTemplate的交易执行

### 数据库扩展

- `etf_rotation_signal` - 轮动信号记录表
- `etf_rotation_position` - 持仓记录表
- `etf_risk_event` - 风险事件记录表

### 不涉及变更

- 现有ETF数据获取模块 (`fund_etf_em.py`) - 直接复用
- 现有指标计算模块 (`calculate_indicator.py`) - 直接复用
- 现有交易框架 (`trade/robot/`) - 扩展使用，不做破坏性修改

## Capabilities

### New Capabilities

- `etf-data-skill`: ETF数据获取能力 - 封装实时行情、历史K线、溢价率等数据获取，供AI Agent调用
- `momentum-analyzer-skill`: 动量分析能力 - 21日动量计算、Z-Score标准化、50日均线趋势过滤、信号生成
- `risk-control-skill`: 风险控制能力 - 止损监控、回撤预警、波动率择时、仓位控制建议
- `market-context-skill`: 市场环境感知能力 - 新闻解读、政策风险识别、宏观数据获取、市场状态评估
- `rebalance-executor-skill`: 调仓执行能力 - 交易指令生成、订单执行、滑点控制、执行日志
- `rotation-orchestrator`: 轮动策略编排能力 - Agent调度、规则加载、综合决策、人工干预接口

### Modified Capabilities

无现有capability被修改。所有新增能力均为独立模块，与现有系统松耦合。

## Impact

### 代码影响

| 模块 | 影响程度 | 说明 |
|-----|---------|------|
| `instock/core/crawling/` | 无 | 直接调用现有函数 |
| `instock/core/indicator/` | 无 | 直接调用现有函数 |
| `instock/trade/robot/` | 低 | 新增策略文件，不修改框架代码 |
| `instock/core/tablestructure.py` | 低 | 仅新增表定义 |

### 依赖影响

- 无新增外部依赖
- 复用现有依赖：pandas, numpy, talib, requests, easytrader

### 系统影响

- 与现有选股策略并行运行，互不干扰
- 可作为独立策略启动，不影响其他功能
- 数据库新增3张表，存储空间需求小幅增加

### 运行影响

- 每日调仓前执行一次完整分析流程
- 交易时段持续监控风险状态
- 预估单次完整分析耗时 < 30秒

---

> 详细技术设计见 `design.md`，各Skill规格见 `specs/<capability>/spec.md`
