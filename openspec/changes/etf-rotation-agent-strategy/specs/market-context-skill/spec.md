# Market Context Skill Specification

## Overview

市场环境感知能力，提供新闻解读、政策风险识别、宏观数据获取、市场状态评估等AI增强功能。该Skill仅在实盘模式下运行，回测模式自动跳过。

---

## ADDED Requirements

### Requirement: 市场状态评估

系统SHALL提供整体市场状态评估能力。

#### Scenario: 获取市场状态

- **WHEN** 调用 `assess_market_status()` 在实盘模式下
- **THEN** 系统返回市场状态评估：
  ```python
  {
      "overall_status": "BULLISH" | "BEARISH" | "NEUTRAL" | "VOLATILE",
      "confidence": 0.75,
      "indicators": {
          "market_breadth": 0.65,  # 上涨股票占比
          "volume_trend": "INCREASING",
          "volatility_level": "MODERATE"
      },
      "key_factors": ["科技股领涨", "成交量放大"],
      "generated_at": "2024-01-15 14:30:00"
  }
  ```

#### Scenario: 回测模式跳过

- **WHEN** 调用 `assess_market_status()` 在回测模式下
- **THEN** 系统返回默认值：
  ```python
  {
      "overall_status": "NEUTRAL",
      "confidence": 0,
      "source": "BACKTEST_DEFAULT",
      "note": "市场环境感知在回测模式下不可用"
  }
  ```

---

### Requirement: 政策风险识别

系统SHALL提供政策风险识别能力。

#### Scenario: 检测政策风险

- **WHEN** 调用 `detect_policy_risk()` 在实盘模式下
- **THEN** 系统返回政策风险评估：
  ```python
  {
      "risk_level": "HIGH" | "MEDIUM" | "LOW",
      "detected_events": [
          {
              "event_type": "REGULATORY_CHANGE",
              "description": "证监会发布新规",
              "affected_sectors": ["金融科技", "互联网"],
              "potential_impact": "NEGATIVE"
          }
      ],
      "recommendation": "规避金融科技ETF，关注低风险品种"
  }
  ```

#### Scenario: 无政策风险事件

- **WHEN** 近期无重大政策事件
- **THEN** 系统返回低风险状态：
  ```python
  {
      "risk_level": "LOW",
      "detected_events": [],
      "recommendation": "正常投资策略"
  }
  ```

---

### Requirement: 新闻情绪分析

系统SHALL提供财经新闻情绪分析能力。

#### Scenario: 分析市场新闻情绪

- **WHEN** 调用 `analyze_news_sentiment(topics=['A股', 'ETF', '宏观经济'])` 在实盘模式下
- **THEN** 系统返回新闻情绪分析：
  ```python
  {
      "sentiment_score": 0.35,  # -1 到 1，正值偏乐观
      "sentiment_label": "SLIGHTLY_BULLISH",
      "key_news": [
          {
              "title": "央行降准0.5个百分点",
              "sentiment": 0.8,
              "relevance": 0.9
          }
      ],
      "topic_sentiment": {
          "A股": 0.4,
          "ETF": 0.3,
          "宏观经济": 0.5
      }
  }
  ```

#### Scenario: 极端情绪预警

- **WHEN** 新闻情绪达到极端值 (> 0.8 或 < -0.8)
- **THEN** 系统返回极端情绪预警：
  ```python
  {
      "alert": "EXTREME_SENTIMENT",
      "direction": "TOO_BULLISH",
      "score": 0.85,
      "warning": "市场情绪过于乐观，注意回调风险"
  }
  ```

---

### Requirement: 宏观数据获取

系统SHALL提供关键宏观数据的获取和分析能力。

#### Scenario: 获取宏观经济指标

- **WHEN** 调用 `get_macro_indicators()` 在实盘模式下
- **THEN** 系统返回关键宏观指标：
  ```python
  {
      "gdp_growth": {
          "value": 5.2,
          "period": "2023Q4",
          "trend": "STABLE"
      },
      "cpi": {
          "value": 0.3,
          "period": "2024-01",
          "trend": "LOW"
      },
      "pmi": {
          "value": 49.2,
          "period": "2024-01",
          "trend": "CONTRACTION"
      },
      "interest_rate": {
          "value": 3.45,
          "trend": "STABLE"
      },
      "assessment": "经济温和复苏，通胀低位"
  }
  ```

#### Scenario: 宏观数据与资产配置建议

- **WHEN** 宏观数据显示特定经济周期
- **THEN** 系统返回资产配置建议：
  ```python
  {
      "economic_cycle": "RECOVERY",
      "recommended_allocation": {
          "股票型ETF": 0.60,
          "债券型ETF": 0.30,
          "商品型ETF": 0.10
      },
      "reasoning": "经济复苏期，权益类资产占优"
  }
  ```

---

### Requirement: 行业轮动分析

系统SHALL提供行业轮动趋势分析能力。

#### Scenario: 分析行业资金流向

- **WHEN** 调用 `analyze_sector_rotation()` 在实盘模式下
- **THEN** 系统返回行业轮动分析：
  ```python
  {
      "hot_sectors": ["科技", "新能源", "医药"],
      "cold_sectors": ["地产", "银行"],
      "rotation_signal": {
          "direction": "GROWTH_TO_VALUE",
          "strength": 0.6
      },
      "etf_implications": {
          "favor": ["159915 创业板ETF", "512760 半导体ETF"],
          "avoid": ["512200 房地产ETF"]
      }
  }
  ```

---

### Requirement: 市场环境综合报告

系统SHALL生成市场环境综合报告。

#### Scenario: 生成综合报告

- **WHEN** 调用 `generate_market_report()` 在实盘模式下
- **THEN** 系统返回综合报告：
  ```python
  {
      "report_date": "2024-01-15",
      "summary": "市场整体偏乐观，科技板块领涨",
      "market_status": {...},
      "policy_risk": {...},
      "news_sentiment": {...},
      "macro_indicators": {...},
      "sector_rotation": {...},
      "action_recommendations": [
          "维持科技ETF配置",
          "规避政策敏感板块",
          "关注成交量变化"
      ],
      "confidence": 0.7
  }
  ```

---

### Requirement: 市场环境与信号融合

系统SHAL提供将市场环境信息与交易信号融合的能力。

#### Scenario: 融合市场环境调整信号

- **WHEN** 调用 `enhance_signal(base_signal, market_context)` 在实盘模式下
- **THEN** 系统返回增强后的信号：
  ```python
  {
      "original_signal": "STRONG_BUY",
      "adjusted_signal": "WEAK_BUY",
      "adjustments": [
          {
              "factor": "policy_risk",
              "impact": -1,  # 降低信号强度
              "reason": "行业政策风险上升"
          },
          {
              "factor": "extreme_sentiment",
              "impact": -0.5,
              "reason": "市场情绪过热"
          }
      ],
      "final_confidence": 0.55
  }
  ```

#### Scenario: 市场环境支持原信号

- **WHEN** 市场环境与原信号方向一致
- **THEN** 系统增强信号强度：
  ```python
  {
      "original_signal": "STRONG_BUY",
      "adjusted_signal": "STRONG_BUY",
      "adjustments": [
          {
              "factor": "market_breadth",
              "impact": +0.3,
              "reason": "市场广度健康"
          }
      ],
      "final_confidence": 0.85
  }
  ```

---

### Requirement: 异常市场事件检测

系统SHALL提供异常市场事件检测能力。

#### Scenario: 检测市场异常

- **WHEN** 调用 `detect_market_anomalies()` 在实盘模式下
- **THEN** 系统返回异常事件列表：
  ```python
  {
      "has_anomaly": True,
      "anomalies": [
          {
              "type": "VOLATILITY_SPIKE",
              "description": "VIX飙升超过30%",
              "severity": "HIGH",
              "recommendation": "降低仓位，等待市场稳定"
          },
          {
              "type": "CORRELATION_BREAKDOWN",
              "description": "股债相关性异常",
              "severity": "MEDIUM"
          }
      ]
  }
  ```

#### Scenario: 无异常事件

- **WHEN** 市场运行正常
- **THEN** 系统返回：
  ```python
  {
      "has_anomaly": False,
      "anomalies": [],
      "last_check": "2024-01-15 14:30:00"
  }
  ```

---

### Requirement: AI决策日志

系统SHALL记录所有AI决策过程，用于事后审计。

#### Scenario: 记录AI决策

- **WHEN** AI参与信号调整或建议生成
- **THEN** 系统记录决策日志：
  ```python
  {
      "log_id": "ai_20240115_143000",
      "decision_type": "SIGNAL_ENHANCEMENT",
      "input": {...},
      "output": {...},
      "reasoning": "政策风险导致降级",
      "model_version": "gpt-4-turbo",
      "timestamp": "2024-01-15 14:30:00"
  }
  ```
