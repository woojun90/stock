# Rebalance Executor Skill Specification

## Overview

调仓执行能力，负责交易指令生成、订单执行、滑点控制、执行日志记录等功能。该Skill与券商交易接口对接，执行实际的买卖操作。

---

## ADDED Requirements

### Requirement: 生成调仓计划

系统SHALL根据信号和持仓生成调仓计划。

#### Scenario: 生成买入计划

- **WHEN** 调用 `generate_rebalance_plan(signals, current_positions, available_cash)` 且有买入信号
- **THEN** 系统返回调仓计划：
  ```python
  {
      "plan_id": "rebalance_20240115",
      "actions": [
          {
              "action": "BUY",
              "etf_code": "510300",
              "etf_name": "沪深300ETF",
              "target_amount": 30000,
              "target_shares": 6000,
              "reason": "动量信号排名#1"
          }
      ],
      "total_buy_amount": 30000,
      "total_sell_amount": 0,
      "estimated_cash_after": 5000
  }
  ```

#### Scenario: 生成卖出计划

- **WHEN** 持仓ETF信号转弱或触发止损
- **THEN** 系统返回卖出计划：
  ```python
  {
      "actions": [
          {
              "action": "SELL",
              "etf_code": "159915",
              "current_shares": 5000,
              "target_shares": 0,
              "reason": "趋势转向，信号转弱"
          }
      ]
  }
  ```

#### Scenario: 资金不足时按优先级分配

- **WHEN** 可用资金不足以买入所有信号ETF
- **THEN** 系统按信号强度排序，优先买入排名靠前的ETF

---

### Requirement: 订单生成

系统SHALL生成可执行的交易订单。

#### Scenario: 生成限价单

- **WHEN** 调用 `create_order(action, etf_code, shares, order_type='LIMIT')`
- **THEN** 系统生成限价单：
  ```python
  {
      "order_id": "order_20240115_001",
      "action": "BUY",
      "etf_code": "510300",
      "shares": 6000,
      "order_type": "LIMIT",
      "limit_price": 5.01,  # 略高于当前价
      "current_price": 5.00,
      "valid_until": "2024-01-15 14:59:00",
      "status": "PENDING"
  }
  ```

#### Scenario: 生成市价单

- **WHEN** 调用 `create_order(action, etf_code, shares, order_type='MARKET')` 且需要立即成交
- **THEN** 系统生成市价单：
  ```python
  {
      "order_type": "MARKET",
      "note": "市价单，以最优价格成交"
  }
  ```

#### Scenario: 订单数量调整

- **WHEN** 订单数量不符合交易规则（如必须为100股整数倍）
- **THEN** 系统自动调整为合规数量

---

### Requirement: 订单执行

系统SHALL执行生成的交易订单。

#### Scenario: 执行买单

- **WHEN** 调用 `execute_order(order)` 且交易接口可用
- **THEN** 系统提交订单并返回结果：
  ```python
  {
      "order_id": "order_20240115_001",
      "execution_status": "FILLED",
      "filled_shares": 6000,
      "filled_price": 5.01,
      "filled_amount": 30060,
      "commission": 15.03,
      "execution_time": "2024-01-15 14:55:30"
  }
  ```

#### Scenario: 部分成交

- **WHEN** 订单部分成交
- **THEN** 系统返回部分成交状态：
  ```python
  {
      "execution_status": "PARTIALLY_FILLED",
      "filled_shares": 3000,
      "remaining_shares": 3000,
      "action": "CANCEL_REMAINING"  # 收盘前取消剩余
  }
  ```

#### Scenario: 执行失败处理

- **WHEN** 订单执行失败（如涨跌停、资金不足）
- **THEN** 系统返回失败信息：
  ```python
  {
      "execution_status": "FAILED",
      "error_code": "INSUFFICIENT_FUNDS",
      "error_message": "可用资金不足",
      "retry_allowed": False
  }
  ```

---

### Requirement: 滑点控制

系统SHALL实现滑点控制机制。

#### Scenario: 计算可接受滑点

- **WHEN** 调用 `calculate_slippage_limit(etf_code, action)`
- **THEN** 系统返回滑点限制：
  ```python
  {
      "etf_code": "510300",
      "action": "BUY",
      "current_price": 5.00,
      "max_slippage_pct": 0.5,  # 最大滑点0.5%
      "max_price": 5.025,  # 最高买入价
      "min_price": null
  }
  ```

#### Scenario: 滑点超限预警

- **WHEN** 执行价格超出滑点限制
- **THEN** 系统返回预警并询问是否继续：
  ```python
  {
      "alert": "SLIPPAGE_EXCEEDED",
      "current_price": 5.03,
      "limit_price": 5.025,
      "slippage_pct": 0.6,
      "action_required": "CONFIRM_OR_CANCEL"
  }
  ```

---

### Requirement: 执行时间窗口

系统SHALL在规定的时间窗口内执行调仓。

#### Scenario: 在调仓窗口内执行

- **WHEN** 当前时间在调仓窗口内（默认14:54-14:58）
- **THEN** 系统允许执行调仓

#### Scenario: 超出调仓窗口

- **WHEN** 当前时间超出调仓窗口
- **THEN** 系统拒绝执行并返回：
  ```python
  {
      "error": "OUTSIDE_TRADING_WINDOW",
      "current_time": "15:05:00",
      "valid_window": "14:54:00 - 14:58:00"
  }
  ```

#### Scenario: 提前准备信号

- **WHEN** 时间到达14:30
- **THEN** 系统开始准备调仓所需的数据和信号

---

### Requirement: 执行日志记录

系统SHALL记录所有调仓执行日志。

#### Scenario: 记录执行日志

- **WHEN** 调仓执行完成
- **THEN** 系统记录日志：
  ```python
  {
      "log_id": "exec_20240115_145530",
      "plan_id": "rebalance_20240115",
      "action": "BUY",
      "etf_code": "510300",
      "shares": 6000,
      "expected_price": 5.00,
      "actual_price": 5.01,
      "slippage": 0.2,  # %
      "commission": 15.03,
      "status": "SUCCESS",
      "execution_time_ms": 350,
      "created_at": "2024-01-15 14:55:30"
  }
  ```

---

### Requirement: 持仓更新

系统SHALL在交易完成后更新持仓记录。

#### Scenario: 更新持仓表

- **WHEN** 交易成功执行
- **THEN** 系统更新 `etf_rotation_position` 表：
  ```python
  {
      "position_date": "2024-01-15",
      "etf_code": "510300",
      "shares": 6000,
      "cost_price": 5.01,
      "market_value": 30060,
      "weight": 0.30,
      "updated_at": "2024-01-15 14:55:30"
  }
  ```

#### Scenario: 卖出后清空持仓

- **WHEN** 卖出全部持仓
- **THEN** 系统将该ETF持仓记录标记为 `closed`

---

### Requirement: 回测模式模拟执行

系统SHALL在回测模式下模拟执行交易。

#### Scenario: 回测模拟成交

- **WHEN** 在回测模式下调用 `execute_order(order)`
- **THEN** 系统使用历史收盘价模拟成交：
  ```python
  {
      "execution_status": "SIMULATED",
      "filled_price": 5.00,  # 使用当日收盘价
      "note": "回测模拟成交，使用收盘价"
  }
  ```

#### Scenario: 回测滑点模拟

- **WHEN** 回测配置启用滑点模拟
- **THEN** 系统在成交价上加入随机滑点（默认0.1%-0.3%）

---

### Requirement: 手动确认机制

系统SHALL支持关键决策的人工确认。

#### Scenario: 大额交易需确认

- **WHEN** 单笔交易金额超过阈值（默认5万元）
- **THEN** 系统暂停执行，等待人工确认：
  ```python
  {
      "status": "PENDING_CONFIRMATION",
      "order": {...},
      "confirmation_timeout": 300,  # 5分钟
      "confirm_url": "/api/confirm/order_xxx"
  }
  ```

#### Scenario: AI建议需确认

- **WHEN** AI调整了交易信号（如从买入改为观望）
- **THEN** 系统记录AI建议并通知用户确认

#### Scenario: 确认超时处理

- **WHEN** 人工确认超时
- **THEN** 系统取消该订单并记录日志

---

### Requirement: 异常恢复

系统SHALL支持执行异常后的恢复机制。

#### Scenario: 订单状态同步

- **WHEN** 系统重启后
- **THEN** 系统自动同步未完成订单的状态

#### Scenario: 断线重连

- **WHEN** 与券商接口断开连接
- **THEN** 系统自动重连并重新提交未执行订单

#### Scenario: 执行回滚

- **WHEN** 调仓过程中发生严重错误
- **THEN** 系统尝试回滚已执行的部分操作
