# Risk Control Skill Specification

## Overview

风险控制能力，实现止损监控、回撤预警、波动率择时、仓位控制建议等功能。该Skill在回测和实盘模式下均生效，保护策略资金安全。

---

## ADDED Requirements

### Requirement: 止损监控

系统SHALL提供止损监控能力，当持仓亏损达到阈值时触发预警。

#### Scenario: 持仓亏损触发止损

- **WHEN** 调用 `check_stop_loss(position, current_price)` 且持仓亏损超过止损线
- **THEN** 系统返回止损信号：
  ```python
  {
      "action": "STOP_LOSS",
      "etf_code": "510300",
      "cost_price": 4.50,
      "current_price": 4.05,
      "loss_pct": -10.0,
      "threshold": -8.0,
      "urgency": "HIGH"
  }
  ```

#### Scenario: 使用ATR动态止损

- **WHEN** 配置启用ATR动态止损 (`use_atr_stop=True`)
- **THEN** 系统根据ATR计算止损位：
  ```
  stop_price = entry_price - (atr * atr_multiplier)
  ```

#### Scenario: 止损后冷却期

- **WHEN** ETF触发止损后
- **THEN** 系统在 `cool_down_days` (默认5个交易日) 内不生成该ETF的买入信号

---

### Requirement: 回撤监控

系统SHALL提供策略级别回撤监控能力。

#### Scenario: 监控策略总回撤

- **WHEN** 调用 `check_drawdown(portfolio_value, peak_value)` 且回撤超过阈值
- **THEN** 系统返回回撤预警：
  ```python
  {
      "alert_type": "DRAWDOWN_WARNING",
      "current_drawdown": -12.5,
      "threshold": -10.0,
      "peak_value": 100000,
      "current_value": 87500,
      "recommended_action": "REDUCE_POSITION"
  }
  ```

#### Scenario: 回撤分级预警

- **WHEN** 回撤达到不同级别
- **THEN** 系统返回对应级别的预警：
  - Level 1 (-10%): 警告，建议减仓
  - Level 2 (-15%): 严重，强制减仓至50%
  - Level 3 (-20%): 危急，强制清仓

#### Scenario: 回撤恢复后重置

- **WHEN** 策略净值创新高
- **THEN** 系统重置峰值并清除回撤预警状态

---

### Requirement: 波动率择时

系统SHALL提供基于波动率的择时能力，在高波动环境下降低仓位。

#### Scenario: 高波动环境降仓

- **WHEN** 调用 `check_volatility(etf_code)` 且ATR相对值超过阈值
- **THEN** 系统返回降仓建议：
  ```python
  {
      "action": "REDUCE_VOLATILITY",
      "etf_code": "159915",
      "atr_relative": 0.035,  # ATR / Price
      "threshold": 0.025,
      "recommended_position": 0.5,  # 建议仓位50%
      "reason": "high_volatility"
  }
  ```

#### Scenario: 计算波动率分位数

- **WHEN** 调用 `get_volatility_percentile(etf_code, lookback=252)`
- **THEN** 系统返回当前ATR在历史中的分位数：
  ```python
  {
      "percentile": 85.5,  # 当前ATR高于85.5%的历史值
      "current_atr": 0.15,
      "historical_median": 0.10,
      "status": "ELEVATED"
  }
  ```

---

### Requirement: 仓位控制

系统SHALL提供仓位控制建议能力。

#### Scenario: 计算建议仓位

- **WHEN** 调用 `calculate_position_size(signal_strength, volatility, drawdown_status)`
- **THEN** 系统返回建议仓位：
  ```python
  {
      "base_position": 0.3,  # 基础仓位30%
      "signal_adjusted": 0.35,  # 信号强度调整后
      "volatility_adjusted": 0.28,  # 波动率调整后
      "final_position": 0.28,
      "adjustments": [
          {"factor": "signal", "impact": +0.05},
          {"factor": "volatility", "impact": -0.07}
      ]
  }
  ```

#### Scenario: 最大仓位限制

- **WHEN** 计算的仓位超过配置的最大仓位
- **THEN** 系统将仓位限制在 `max_position_pct` (默认40%)

#### Scenario: 单一ETF仓位限制

- **WHEN** 单个ETF建议仓位超过阈值
- **THEN** 系统限制单一ETF仓位不超过 `max_single_etf_pct` (默认30%)

---

### Requirement: 风险事件记录

系统SHALL记录所有风险事件，用于后续分析。

#### Scenario: 记录风险事件

- **WHEN** 触发风险预警或风控动作
- **THEN** 系统将事件存入数据库表 `etf_risk_event`：
  ```python
  {
      "event_date": "2024-01-15",
      "event_type": "STOP_LOSS" | "DRAWDOWN_WARNING" | "VOLATILITY_HIGH",
      "etf_code": "510300",
      "severity": "HIGH" | "MEDIUM" | "LOW",
      "details": {...},
      "action_taken": "SOLD_1000_SHARES",
      "created_at": "2024-01-15 14:35:00"
  }
  ```

---

### Requirement: 风险评分

系统SHALL提供整体风险评分能力。

#### Scenario: 计算组合风险评分

- **WHEN** 调用 `calculate_risk_score(portfolio, market_data)`
- **THEN** 系统返回综合风险评分：
  ```python
  {
      "overall_score": 65,  # 0-100，越高风险越大
      "components": {
          "drawdown_risk": 70,
          "volatility_risk": 55,
          "concentration_risk": 60
      },
      "recommendation": "MODERATE_RISK",
      "suggested_actions": ["REDUCE_TECH_EXPOSURE"]
  }
  ```

#### Scenario: 集中度风险检测

- **WHEN** 某一类ETF持仓占比过高
- **THEN** 系统返回集中度风险警告：
  ```python
  {
      "alert_type": "CONCENTRATION_RISK",
      "category": "股票型ETF",
      "current_weight": 0.85,
      "threshold": 0.70,
      "recommendation": "增加债券型或商品型ETF配置"
  }
  ```

---

### Requirement: 风控阈值配置

系统SHALL支持风控阈值的配置化管理。

#### Scenario: 加载风控配置

- **WHEN** 调用 `load_risk_config(config_path='config/risk_thresholds.yaml')`
- **THEN** 系统加载配置：
  ```yaml
  stop_loss:
    fixed_pct: -0.08
    use_atr: true
    atr_multiplier: 2.0
  drawdown:
    level_1: -0.10
    level_2: -0.15
    level_3: -0.20
  position:
    max_total: 0.95
    max_single: 0.30
    min_cash_reserve: 0.05
  ```

#### Scenario: 动态调整阈值

- **WHEN** 运行时更新风控配置
- **THEN** 系统在下一个交易日应用新阈值，并记录变更日志

---

### Requirement: 风控状态查询

系统SHALL提供当前风控状态的查询能力。

#### Scenario: 查询当前风控状态

- **WHEN** 调用 `get_risk_status()`
- **THEN** 系统返回当前风控状态：
  ```python
  {
      "status": "NORMAL" | "WARNING" | "CRITICAL",
      "active_alerts": [...],
      "cooldown_etfs": ["159915"],  # 冷却期内的ETF
      "position_limits": {...},
      "last_check": "2024-01-15 14:30:00"
  }
  ```

#### Scenario: 清除过期的风控状态

- **WHEN** 冷却期结束或风险解除
- **THEN** 系统自动清除相关风控状态
