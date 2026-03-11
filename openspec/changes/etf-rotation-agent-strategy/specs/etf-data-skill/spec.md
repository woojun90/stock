# ETF Data Skill Specification

## Overview

ETF数据获取能力，封装东方财富网ETF数据的获取接口，提供实时行情、历史K线、溢价率等数据，供AI Agent和其他Skills调用。

---

## ADDED Requirements

### Requirement: 获取ETF实时行情

系统SHALL提供获取ETF实时行情数据的能力，包括最新价、涨跌幅、成交量、成交额等。

#### Scenario: 获取单个ETF实时行情

- **WHEN** 调用 `get_realtime_quote(etf_code)` 传入有效的ETF代码
- **THEN** 系统返回该ETF的实时行情数据，包含：
  - `code`: ETF代码
  - `name`: ETF名称
  - `price`: 最新价
  - `change_pct`: 涨跌幅(%)
  - `volume`: 成交量(手)
  - `amount`: 成交额(元)
  - `high`: 最高价
  - `low`: 最低价
  - `open`: 开盘价
  - `prev_close`: 昨收价
  - `timestamp`: 数据时间戳

#### Scenario: 批量获取ETF实时行情

- **WHEN** 调用 `get_realtime_quotes(etf_codes)` 传入ETF代码列表
- **THEN** 系统返回所有ETF的实时行情数据列表

#### Scenario: ETF代码无效

- **WHEN** 调用 `get_realtime_quote(etf_code)` 传入无效或不存在的ETF代码
- **THEN** 系统返回错误信息，包含错误码 `INVALID_ETF_CODE`

---

### Requirement: 获取ETF历史K线数据

系统SHALL提供获取ETF历史K线数据的能力，支持多种周期（日K、周K、月K）。

#### Scenario: 获取日K线数据

- **WHEN** 调用 `get_kline(etf_code, period='daily', start_date, end_date)` 指定日期范围
- **THEN** 系统返回该时间段内的日K线数据，每条记录包含：
  - `date`: 交易日期
  - `open`: 开盘价
  - `high`: 最高价
  - `low`: 最低价
  - `close`: 收盘价
  - `volume`: 成交量
  - `amount`: 成交额

#### Scenario: 获取最近N个交易日K线

- **WHEN** 调用 `get_kline(etf_code, period='daily', count=21)` 指定数量
- **THEN** 系统返回最近21个交易日的K线数据

#### Scenario: 数据不足时返回可用数据

- **WHEN** 请求的K线数据超过ETF上市以来的交易日数量
- **THEN** 系统返回实际可用的所有K线数据，并在响应中标注 `available_count`

---

### Requirement: 获取ETF基本信息

系统SHALL提供获取ETF基本信息的能力，包括基金规模、跟踪指数、管理费率等。

#### Scenario: 获取ETF基本信息

- **WHEN** 调用 `get_etf_info(etf_code)`
- **THEN** 系统返回ETF基本信息，包含：
  - `code`: ETF代码
  - `name`: ETF名称
  - `type`: ETF类型（股票型、债券型、商品型、货币型）
  - `track_index`: 跟踪指数代码
  - `track_index_name`: 跟踪指数名称
  - `management_fee`: 管理费率
  - `custody_fee`: 托管费率
  - `scale`: 基金规模(亿元)
  - `listing_date`: 上市日期

---

### Requirement: 获取ETF溢价率数据

系统SHALL提供获取ETF溢价率数据的能力，用于评估ETF定价偏离程度。

#### Scenario: 获取实时溢价率

- **WHEN** 调用 `get_premium_rate(etf_code)`
- **THEN** 系统返回ETF的实时溢价率数据，包含：
  - `code`: ETF代码
  - `nav`: 基金净值
  - `nav_date`: 净值日期
  - `price`: 市场价格
  - `premium_rate`: 溢价率(%)
  - `premium_amount`: 溢价金额(元)

#### Scenario: 溢价率数据延迟时返回缓存数据

- **WHEN** 净值数据更新滞后（T日收盘后净值未更新）
- **THEN** 系统返回前一日净值计算的溢价率，并标注 `data_status: delayed`

---

### Requirement: 获取ETF资产池列表

系统SHALL提供获取配置的ETF资产池列表的能力。

#### Scenario: 获取ETF资产池

- **WHEN** 调用 `get_etf_pool(pool_name='default')`
- **THEN** 系统返回配置文件中定义的ETF列表，包含：
  - `pool_name`: 资产池名称
  - `etf_list`: ETF代码列表
  - `categories`: 分类信息（如：大盘、小盘、商品、债券）

#### Scenario: 资产池配置文件不存在

- **WHEN** 调用 `get_etf_pool(pool_name='nonexistent')`
- **THEN** 系统返回错误信息，错误码 `POOL_NOT_FOUND`

---

### Requirement: 数据缓存机制

系统SHALL实现数据缓存机制，避免重复请求同一数据。

#### Scenario: 缓存命中

- **WHEN** 在缓存有效期内重复请求相同数据
- **THEN** 系统直接返回缓存数据，不发起网络请求

#### Scenario: 缓存过期

- **WHEN** 请求的数据缓存已过期
- **THEN** 系统重新获取数据并更新缓存

#### Scenario: 强制刷新

- **WHEN** 调用时指定 `force_refresh=True`
- **THEN** 系统忽略缓存，重新获取最新数据

---

### Requirement: 错误处理与重试

系统SHALL实现错误处理和自动重试机制。

#### Scenario: 网络请求失败自动重试

- **WHEN** 数据请求因网络问题失败
- **THEN** 系统自动重试最多3次，每次间隔递增（1秒、2秒、4秒）

#### Scenario: 重试失败后返回错误

- **WHEN** 重试3次后仍无法获取数据
- **THEN** 系统返回错误信息，错误码 `DATA_FETCH_FAILED`，包含最后一次错误详情

---

### Requirement: 数据返回格式统一

系统SHALL统一所有数据返回格式，便于调用方处理。

#### Scenario: 成功响应格式

- **WHEN** 数据获取成功
- **THEN** 返回格式为：
  ```python
  {
      "success": True,
      "data": [...],  # 实际数据
      "timestamp": "2024-01-15 14:30:00",
      "source": "eastmoney"
  }
  ```

#### Scenario: 失败响应格式

- **WHEN** 数据获取失败
- **THEN** 返回格式为：
  ```python
  {
      "success": False,
      "error_code": "ERROR_CODE",
      "error_message": "详细错误信息",
      "timestamp": "2024-01-15 14:30:00"
  }
  ```
