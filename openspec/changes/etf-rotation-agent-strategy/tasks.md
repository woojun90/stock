# ETF轮动策略 - 实现任务清单

## 1. 项目基础设施

- [ ] 1.1 创建 `instock/skills/` 目录结构
- [ ] 1.2 创建 Skills 模块 `__init__.py` 文件
- [ ] 1.3 创建 `config/` 配置目录
- [ ] 1.4 创建数据库表结构扩展 (`tablestructure.py` 新增3张表)

## 2. ETF数据获取Skill (etf-data-skill)

- [ ] 2.1 实现 `get_realtime_quote()` 单个ETF实时行情获取
- [ ] 2.2 实现 `get_realtime_quotes()` 批量ETF实时行情获取
- [ ] 2.3 实现 `get_kline()` 历史K线数据获取
- [ ] 2.4 实现 `get_etf_info()` ETF基本信息获取
- [ ] 2.5 实现 `get_premium_rate()` ETF溢价率数据获取
- [ ] 2.6 实现 `get_etf_pool()` ETF资产池列表获取
- [ ] 2.7 实现数据缓存机制
- [ ] 2.8 实现错误处理与重试机制
- [ ] 2.9 统一数据返回格式封装
- [ ] 2.10 编写单元测试

## 3. 动量分析Skill (momentum-analyzer-skill)

- [ ] 3.1 实现 `calculate_momentum()` 动量值计算
- [ ] 3.2 实现 `calculate_zscore()` Z-Score标准化
- [ ] 3.3 实现 `is_above_ma()` 趋势过滤判断
- [ ] 3.4 实现 `get_ma_deviation()` 均线偏离度计算
- [ ] 3.5 实现 `generate_signal()` 轮动信号生成
- [ ] 3.6 实现 `rank_etf_pool()` ETF池排序
- [ ] 3.7 实现 `record_signal()` 信号历史记录
- [ ] 3.8 实现回测模式支持
- [ ] 3.9 实现参数配置化加载与验证
- [ ] 3.10 编写单元测试

## 4. 风险控制Skill (risk-control-skill)

- [ ] 4.1 实现 `check_stop_loss()` 止损监控
- [ ] 4.2 实现ATR动态止损计算
- [ ] 4.3 实现止损后冷却期机制
- [ ] 4.4 实现 `check_drawdown()` 回撤监控
- [ ] 4.5 实现回撤分级预警
- [ ] 4.6 实现 `check_volatility()` 波动率择时
- [ ] 4.7 实现 `get_volatility_percentile()` 波动率分位数计算
- [ ] 4.8 实现 `calculate_position_size()` 仓位控制建议
- [ ] 4.9 实现最大仓位限制与单ETF仓位限制
- [ ] 4.10 实现风险事件记录
- [ ] 4.11 实现 `calculate_risk_score()` 综合风险评分
- [ ] 4.12 实现集中度风险检测
- [ ] 4.13 实现风控阈值配置加载
- [ ] 4.14 实现 `get_risk_status()` 风控状态查询
- [ ] 4.15 编写单元测试

## 5. 市场环境感知Skill (market-context-skill)

- [ ] 5.1 实现 `assess_market_status()` 市场状态评估
- [ ] 5.2 实现回测模式跳过逻辑
- [ ] 5.3 实现 `detect_policy_risk()` 政策风险识别
- [ ] 5.4 实现 `analyze_news_sentiment()` 新闻情绪分析
- [ ] 5.5 实现极端情绪预警
- [ ] 5.6 实现 `get_macro_indicators()` 宏观数据获取
- [ ] 5.7 实现经济周期资产配置建议
- [ ] 5.8 实现 `analyze_sector_rotation()` 行业轮动分析
- [ ] 5.9 实现 `generate_market_report()` 综合报告生成
- [ ] 5.10 实现 `enhance_signal()` 市场环境与信号融合
- [ ] 5.11 实现 `detect_market_anomalies()` 异常市场事件检测
- [ ] 5.12 实现AI决策日志记录
- [ ] 5.13 编写单元测试

## 6. 调仓执行Skill (rebalance-executor-skill)

- [ ] 6.1 实现 `generate_rebalance_plan()` 调仓计划生成
- [ ] 6.2 实现资金不足时优先级分配
- [ ] 6.3 实现 `create_order()` 订单生成（限价单/市价单）
- [ ] 6.4 实现订单数量合规调整
- [ ] 6.5 实现 `execute_order()` 订单执行
- [ ] 6.6 实现部分成交处理
- [ ] 6.7 实现执行失败处理
- [ ] 6.8 实现滑点控制机制
- [ ] 6.9 实现滑点超限预警
- [ ] 6.10 实现执行时间窗口控制
- [ ] 6.11 实现执行日志记录
- [ ] 6.12 实现持仓更新
- [ ] 6.13 实现回测模式模拟执行
- [ ] 6.14 实现手动确认机制
- [ ] 6.15 实现异常恢复机制
- [ ] 6.16 编写单元测试

## 7. 轮动策略编排 (rotation-orchestrator)

- [ ] 7.1 实现 `initialize()` 策略初始化
- [ ] 7.2 实现配置文件缺失默认配置处理
- [ ] 7.3 实现 `schedule_rebalance()` 定时任务调度
- [ ] 7.4 实现 `schedule_risk_monitor()` 风控监控定时任务
- [ ] 7.5 实现 `run_rotation_cycle()` 完整轮动流程
- [ ] 7.6 实现步骤失败处理
- [ ] 7.7 实现 `make_decision()` 综合决策
- [ ] 7.8 实现冲突信号优先级处理
- [ ] 7.9 实现 `load_rules()` 规则配置加载
- [ ] 7.10 实现 `reload_config()` 热更新配置
- [ ] 7.11 实现配置版本管理
- [ ] 7.12 实现 `pause_strategy()` / `resume_strategy()` 人工干预
- [ ] 7.13 实现 `manual_rebalance()` 手动触发调仓
- [ ] 7.14 实现 `override_decision()` 决策覆盖
- [ ] 7.15 实现 `get_status()` 策略状态监控
- [ ] 7.16 实现AI Agent查询接口
- [ ] 7.17 实现AI权限控制
- [ ] 7.18 实现事件通知机制（交易/风险/日报）
- [ ] 7.19 实现决策日志与审计追踪
- [ ] 7.20 实现 `run_backtest()` 回测支持
- [ ] 7.21 编写单元测试

## 8. 规则配置文件

- [ ] 8.1 创建 `config/etf_rotation_rules.yaml` 核心策略规则
- [ ] 8.2 创建 `config/etf_pool.yaml` ETF资产池定义
- [ ] 8.3 创建 `config/risk_thresholds.yaml` 风控阈值配置

## 9. 策略集成与交易框架对接

- [ ] 9.1 创建 `instock/core/strategy/etf_rotation.py` 策略核心模块
- [ ] 9.2 创建 `instock/trade/strategies/etf_rotation_strategy.py` 交易策略入口
- [ ] 9.3 集成到 `MainEngine` 和 `EventEngine`
- [ ] 9.4 实现定时事件触发

## 10. 回测引擎扩展

- [ ] 10.1 创建 `instock/core/backtest/rotation_backtest.py` 轮动回测引擎
- [ ] 10.2 实现历史信号回放
- [ ] 10.3 实现回测绩效计算
- [ ] 10.4 实现回测报告生成

## 11. 集成测试

- [ ] 11.1 编写端到端轮动流程测试
- [ ] 11.2 编写回测验证测试
- [ ] 11.3 编写风控触发测试
- [ ] 11.4 编写AI增强功能测试（模拟模式）

## 12. 文档与部署

- [ ] 12.1 编写Skills API文档
- [ ] 12.2 编写配置说明文档
- [ ] 12.3 编写部署指南
- [ ] 12.4 编写使用示例

---

**预估工时**: 40-60 小时

**关键路径**: 1 → 2 → 3 → 4 → 7 → 9 (基础设施 → 数据 → 动量 → 风控 → 编排 → 集成)

**可并行**: 2/3/4/5/6 可并行开发，7 依赖前述所有Skills
