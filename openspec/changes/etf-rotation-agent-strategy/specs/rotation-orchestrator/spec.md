# Rotation Orchestrator Specification

## Overview

轮动策略编排能力，负责Agent调度、规则加载、综合决策、人工干预接口等核心编排功能。作为策略的大脑，协调各Skills完成完整的轮动流程。

---

## ADDED Requirements

### Requirement: 策略初始化

系统SHALL提供策略初始化能力，加载所有必要的配置和Skills。

#### Scenario: 初始化策略

- **WHEN** 调用 `initialize(config_path='config/etf_rotation_rules.yaml')`
- **THEN** 系统完成初始化：
  ```python
  {
      "status": "INITIALIZED",
      "loaded_config": {
          "momentum_period": 21,
          "ma_period": 50,
          "rebalance_time": "14:54"
      },
      "loaded_skills": [
          "etf_data_skill",
          "momentum_analyzer_skill",
          "risk_control_skill",
          "market_context_skill",
          "rebalance_executor_skill"
      ],
      "etf_pool_count": 15,
      "init_time": "2024-01-15 09:00:00"
  }
  ```

#### Scenario: 配置文件缺失

- **WHEN** 配置文件不存在或格式错误
- **THEN** 系统使用默认配置并返回警告：
  ```python
  {
      "status": "INITIALIZED_WITH_DEFAULTS",
      "warnings": ["配置文件不存在，使用默认配置"]
  }
  ```

---

### Requirement: 定时任务调度

系统SHALL提供定时任务调度能力。

#### Scenario: 注册调仓定时任务

- **WHEN** 调用 `schedule_rebalance(cron='0 14 54 ? * FRI')` 注册每周五14:54的调仓任务
- **THEN** 系统注册定时任务：
  ```python
  {
      "task_id": "rebalance_weekly",
      "cron": "0 14 54 ? * FRI",
      "next_run": "2024-01-19 14:54:00",
      "status": "SCHEDULED"
  }
  ```

#### Scenario: 注册风控监控任务

- **WHEN** 调用 `schedule_risk_monitor(cron='0 9 30-15 0 ? * *')` 注册每日交易时段的风控监控
- **THEN** 系统注册风控监控任务，每隔一段时间检查风险状态

#### Scenario: 任务触发

- **WHEN** 定时任务触发时间到达
- **THEN** 系统执行对应任务并记录执行日志

---

### Requirement: 完整轮动流程

系统SHALL提供执行完整轮动流程的能力。

#### Scenario: 执行完整轮动流程

- **WHEN** 调用 `run_rotation_cycle(mode='live')` 在实盘模式下
- **THEN** 系统按顺序执行：
  ```python
  {
      "cycle_id": "cycle_20240119_145400",
      "steps": [
          {"step": "fetch_data", "status": "SUCCESS", "duration_ms": 1200},
          {"step": "calculate_signals", "status": "SUCCESS", "duration_ms": 800},
          {"step": "assess_risk", "status": "SUCCESS", "duration_ms": 300},
          {"step": "market_context", "status": "SUCCESS", "duration_ms": 2000},
          {"step": "make_decision", "status": "SUCCESS", "duration_ms": 100},
          {"step": "execute_rebalance", "status": "SUCCESS", "duration_ms": 3500}
      ],
      "total_duration_ms": 7900,
      "result": {
          "actions_taken": ["BUY 510300"],
          "final_positions": {...}
      }
  }
  ```

#### Scenario: 步骤失败处理

- **WHEN** 流程中某步骤失败
- **THEN** 系统中止后续步骤，记录失败原因：
  ```python
  {
      "cycle_id": "cycle_20240119_145400",
      "failed_at_step": "fetch_data",
      "error": "DATA_FETCH_FAILED",
      "actions_taken": [],
      "status": "FAILED"
  }
  ```

---

### Requirement: 综合决策

系统SHALL综合各Skills的结果做出最终决策。

#### Scenario: 综合信号和风控决策

- **WHEN** 调用 `make_decision(signals, risk_status, market_context)`
- **THEN** 系统返回综合决策：
  ```python
  {
      "decision_id": "decision_20240119_145500",
      "final_actions": [
          {
              "action": "BUY",
              "etf_code": "510300",
              "amount": 30000,
              "confidence": 0.85,
              "contributing_factors": {
                  "signal_score": 2.1,
                  "risk_status": "LOW",
                  "market_sentiment": 0.3
              }
          }
      ],
      "excluded_actions": [
          {
              "action": "BUY",
              "etf_code": "159915",
              "reason": "风控冷却期",
              "exclusion_source": "risk_control_skill"
          }
      ],
      "decision_rationale": "动量信号强，风险可控，市场环境支持"
  }
  ```

#### Scenario: 冲突信号处理

- **WHEN** 多个信号来源给出冲突建议
- **THEN** 系统按优先级处理：风控 > 市场环境 > 动量信号

---

### Requirement: 规则配置管理

系统SHALL提供规则配置的管理能力。

#### Scenario: 加载规则配置

- **WHEN** 调用 `load_rules(config_path)`
- **THEN** 系统加载并验证规则配置：
  ```python
  {
      "loaded": True,
      "rules": {
          "signal_rules": {...},
          "risk_rules": {...},
          "execution_rules": {...}
      },
      "validation_errors": []
  }
  ```

#### Scenario: 热更新配置

- **WHEN** 调用 `reload_config()` 或检测到配置文件变更
- **THEN** 系统重新加载配置并通知相关Skills：
  ```python
  {
      "reloaded": True,
      "previous_config_hash": "abc123",
      "new_config_hash": "def456",
      "affected_skills": ["momentum_analyzer_skill", "risk_control_skill"],
      "reload_time": "2024-01-15 14:00:00"
  }
  ```

#### Scenario: 配置版本管理

- **WHEN** 配置更新
- **THEN** 系统保留历史配置版本，支持回滚

---

### Requirement: 人工干预接口

系统SHALL提供人工干预的接口。

#### Scenario: 暂停策略

- **WHEN** 调用 `pause_strategy(reason='市场异常波动')`
- **THEN** 系统暂停策略执行：
  ```python
  {
      "status": "PAUSED",
      "pause_time": "2024-01-15 14:30:00",
      "reason": "市场异常波动",
      "scheduled_tasks_paused": ["rebalance_weekly"],
      "resume_command": "resume_strategy()"
  }
  ```

#### Scenario: 恢复策略

- **WHEN** 调用 `resume_strategy()`
- **THEN** 系统恢复策略执行

#### Scenario: 手动触发调仓

- **WHEN** 调用 `manual_rebalance(reason='补仓机会')`
- **THEN** 系统立即执行一次调仓流程，不受定时任务限制

#### Scenario: 覆盖决策

- **WHEN** 调用 `override_decision(decision_id, override_actions)`
- **THEN** 系统使用人工指定的操作替代AI决策：
  ```python
  {
      "original_decision": {...},
      "override_actions": [
          {"action": "HOLD", "etf_code": "510300"}
      ],
      "overridden_by": "manual",
      "override_time": "2024-01-15 14:55:00"
  }
  ```

---

### Requirement: 状态监控

系统SHALL提供策略状态监控能力。

#### Scenario: 获取策略状态

- **WHEN** 调用 `get_status()`
- **THEN** 系统返回当前状态：
  ```python
  {
      "strategy_status": "RUNNING" | "PAUSED" | "STOPPED" | "ERROR",
      "last_rebalance": "2024-01-19 14:54:00",
      "next_rebalance": "2024-01-26 14:54:00",
      "current_positions": [...],
      "pending_orders": [],
      "risk_status": {
          "level": "LOW",
          "active_alerts": []
      },
      "performance": {
          "total_return": 0.15,
          "ytd_return": 0.08,
          "max_drawdown": -0.05
      }
  }
  ```

---

### Requirement: AI Agent接口

系统SHALL提供AI Agent调用的标准接口。

#### Scenario: AI查询策略状态

- **WHEN** AI Agent调用 `query_context(query='当前持仓和风险状态')`
- **THEN** 系统返回结构化上下文：
  ```python
  {
      "query": "当前持仓和风险状态",
      "response": {
          "positions": [...],
          "risk_summary": {...},
          "recent_signals": [...]
      },
      "suggested_actions": ["查看最近的交易日志", "检查风控阈值"]
  }
  ```

#### Scenario: AI请求执行操作

- **WHEN** AI Agent调用 `request_action(action_type, params)` 
- **THEN** 系统验证权限后执行：
  ```python
  {
      "action_requested": "pause_strategy",
      "authorized": True,
      "executed": True,
      "result": {...}
  }
  ```

#### Scenario: AI权限限制

- **WHEN** AI请求超出权限范围的操作（如直接交易）
- **THEN** 系统拒绝并建议人工操作：
  ```python
  {
      "action_requested": "direct_trade",
      "authorized": False,
      "reason": "AI无权直接执行交易，需通过信号生成流程",
      "alternative": "使用generate_signal()生成信号"
  }
  ```

---

### Requirement: 事件通知

系统SHALL提供事件通知机制。

#### Scenario: 发送交易通知

- **WHEN** 调仓执行完成
- **THEN** 系统发送通知：
  ```python
  {
      "notification_type": "REBALANCE_COMPLETE",
      "channel": "wechat" | "email" | "telegram",
      "content": "ETF轮动调仓完成\n买入: 510300 沪深300ETF 6000股\n成交价: 5.01元",
      "timestamp": "2024-01-19 14:56:00"
  }
  ```

#### Scenario: 发送风险预警

- **WHEN** 触发风险预警
- **THEN** 系统立即发送高优先级通知

#### Scenario: 发送日报

- **WHEN** 每日收盘后
- **THEN** 系统发送当日投资日报

---

### Requirement: 日志与审计

系统SHALL提供完整的日志和审计功能。

#### Scenario: 记录决策日志

- **WHEN** 策略做出重要决策
- **THEN** 系统记录详细日志：
  ```python
  {
      "log_type": "DECISION",
      "decision_id": "decision_20240119_145500",
      "inputs": {...},
      "outputs": {...},
      "reasoning": "综合动量信号和市场环境",
      "ai_participation": True,
      "timestamp": "2024-01-19 14:55:00"
  }
  ```

#### Scenario: 审计追踪

- **WHEN** 查询历史决策
- **THEN** 系统返回完整决策链条，从信号生成到执行完成

---

### Requirement: 回测模式支持

系统SHALL支持回测模式运行。

#### Scenario: 启动回测

- **WHEN** 调用 `run_backtest(start_date='2023-01-01', end_date='2024-01-01', initial_capital=100000)`
- **THEN** 系统返回回测结果：
  ```python
  {
      "backtest_id": "bt_20240115_001",
      "period": "2023-01-01 to 2024-01-01",
      "initial_capital": 100000,
      "final_capital": 115000,
      "total_return": 0.15,
      "annualized_return": 0.15,
      "max_drawdown": -0.08,
      "sharpe_ratio": 1.2,
      "win_rate": 0.62,
      "trade_count": 24,
      "trades": [...]
  }
  ```

#### Scenario: 回测与实盘模式切换

- **WHEN** 调用 `set_mode(mode='backtest' | 'live')`
- **THEN** 系统切换运行模式，相应地启用或禁用AI增强功能

#### Scenario: 回测进度报告

- **WHEN** 长时间回测运行中
- **THEN** 系统定期返回进度报告：
  ```python
  {
      "progress": 0.45,
      "current_date": "2023-07-01",
      "trades_so_far": 12,
      "current_capital": 108000
  }
  ```
