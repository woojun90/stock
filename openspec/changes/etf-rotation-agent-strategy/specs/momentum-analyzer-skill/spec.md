# Momentum Analyzer Skill Specification

## Overview

动量分析能力，实现21日动量计算、Z-Score标准化、50日均线趋势过滤、信号生成等核心策略逻辑。该Skill是策略的核心组件，必须在回测和实盘模式下产生一致的确定性结果。

---

## ADDED Requirements

### Requirement: 计算动量值

系统SHALL提供计算ETF动量值的能力，使用收益率计算方法。

#### Scenario: 计算21日动量

- **WHEN** 调用 `calculate_momentum(close_prices, period=21)` 传入收盘价序列
- **THEN** 系统返回动量值序列，计算公式为：
  ```
  momentum = (close_t / close_{t-period}) - 1
  ```

#### Scenario: 数据不足时返回空值

- **WHEN** 输入数据长度小于 `period + 1`
- **THEN** 系统返回空值或部分有效数据，并在元数据中标注 `insufficient_data: True`

---

### Requirement: 动量Z-Score标准化

系统SHALL提供动量值Z-Score标准化的能力，使不同ETF的动量可比较。

#### Scenario: 计算Z-Score

- **WHEN** 调用 `calculate_zscore(momentum_values, lookback_period=252)` 传入动量值序列
- **THEN** 系统返回Z-Score序列，计算公式为：
  ```
  z_score = (momentum - mean(lookback_period)) / std(lookback_period)
  ```

#### Scenario: 标准差为零时返回零值

- **WHEN** lookback_period内的动量值完全相同（标准差为0）
- **THEN** 系统返回Z-Score = 0，避免除零错误

---

### Requirement: 趋势过滤

系统SHALL提供基于移动平均线的趋势过滤能力。

#### Scenario: 判断是否在均线上方

- **WHEN** 调用 `is_above_ma(close_prices, ma_period=50)` 传入收盘价序列
- **THEN** 系统返回布尔值：
  - `True`: 最新收盘价 > 50日均线
  - `False`: 最新收盘价 <= 50日均线

#### Scenario: 计算均线偏离度

- **WHEN** 调用 `get_ma_deviation(close_prices, ma_period=50)`
- **THEN** 系统返回偏离度百分比：
  ```
  deviation = (close - ma50) / ma50 * 100
  ```

#### Scenario: 均线数据不足

- **WHEN** 收盘价数据少于 `ma_period`
- **THEN** 系统返回 `None`，并标注 `reason: insufficient_data`

---

### Requirement: 生成轮动信号

系统SHALL提供综合动量和趋势因素生成轮动信号的能力。

#### Scenario: 生成买入信号

- **WHEN** 调用 `generate_signal(etf_code, z_score, trend_status, thresholds)` 且满足：
  - z_score > thresholds.strong_signal (默认1.0)
  - trend_status = True (在均线上方)
- **THEN** 系统返回强买入信号：
  ```python
  {
      "signal": "STRONG_BUY",
      "score": z_score,
      "trend": "BULLISH",
      "confidence": "HIGH"
  }
  ```

#### Scenario: 生成弱信号

- **WHEN** z_score在 0 和 thresholds.strong_signal 之间，且趋势向上
- **THEN** 系统返回弱买入信号：
  ```python
  {
      "signal": "WEAK_BUY",
      "score": z_score,
      "trend": "BULLISH",
      "confidence": "MEDIUM"
  }
  ```

#### Scenario: 生成卖出信号

- **WHEN** z_score < thresholds.weak_signal (默认0) 或 trend_status = False
- **THEN** 系统返回卖出信号：
  ```python
  {
      "signal": "SELL",
      "score": z_score,
      "trend": "BEARISH" | "NEUTRAL",
      "confidence": "HIGH"
  }
  ```

#### Scenario: 趋势向下时抑制买入

- **WHEN** z_score很高但 trend_status = False (价格在均线下方)
- **THEN** 系统返回观望信号：
  ```python
  {
      "signal": "HOLD",
      "score": z_score,
      "trend": "BEARISH",
      "confidence": "LOW",
      "reason": "trend_filter_failed"
  }
  ```

---

### Requirement: ETF池排序

系统SHALL提供对ETF池按动量信号排序的能力。

#### Scenario: 按动量排序ETF池

- **WHEN** 调用 `rank_etf_pool(etf_pool_data, top_n=3)` 传入ETF池数据
- **THEN** 系统返回按Z-Score降序排列的ETF列表：
  ```python
  {
      "rankings": [
          {"code": "510300", "z_score": 2.5, "signal": "STRONG_BUY", "rank": 1},
          {"code": "159915", "z_score": 1.8, "signal": "STRONG_BUY", "rank": 2},
          ...
      ],
      "generated_at": "2024-01-15 14:30:00"
  }
  ```

#### Scenario: 排除不满足条件的ETF

- **WHEN** 某ETF趋势向下或数据不足
- **THEN** 系统将其排除在排序之外，列入 `excluded` 列表并说明原因

---

### Requirement: 信号历史记录

系统SHALL记录每次生成的信号，用于后续分析和回测验证。

#### Scenario: 记录信号

- **WHEN** 生成信号后调用 `record_signal(signal_data)`
- **THEN** 系统将信号存入数据库表 `etf_rotation_signal`，包含：
  - `signal_date`: 信号日期
  - `etf_code`: ETF代码
  - `signal_type`: 信号类型
  - `z_score`: Z-Score值
  - `trend_status`: 趋势状态
  - `ma50_value`: 50日均线值
  - `momentum_value`: 原始动量值
  - `created_at`: 创建时间

---

### Requirement: 回测模式支持

系统SHALL在回测模式下返回确定性结果。

#### Scenario: 回测模式信号生成

- **WHEN** 调用 `run(mode='backtest', date='2024-01-15')` 指定回测日期
- **THEN** 系统使用该日期的历史数据生成信号，不依赖实时数据

#### Scenario: 回测结果可重现

- **WHEN** 对同一历史日期多次运行回测
- **THEN** 系统每次返回完全相同的信号结果

---

### Requirement: 参数配置化

系统SHALL支持通过配置文件调整策略参数。

#### Scenario: 从配置文件加载参数

- **WHEN** 调用 `load_config(config_path='config/etf_rotation_rules.yaml')`
- **THEN** 系统加载配置文件中的参数：
  ```yaml
  momentum_period: 21
  zscore_lookback: 252
  ma_period: 50
  thresholds:
    strong_signal: 1.0
    weak_signal: 0
  ```

#### Scenario: 参数验证

- **WHEN** 配置参数超出合理范围
- **THEN** 系统返回警告信息并使用默认值：
  - `momentum_period`: 有效范围 [5, 60]
  - `ma_period`: 有效范围 [20, 120]
  - `zscore_lookback`: 有效范围 [60, 500]
