# Prometheus

## 数据存储

### 存储格式

- <metric name>{<label name> = <label value>, ...}

  例如`api_http_requests_total{method="POST", handler="/messages"}`

  - metric name:api_http_requests_total
  - label: 

## 数据模型(Metrics Type)
- `Counter`

- `Gauge`

- `Histogram`

  假设有一个直方图指标`http_request_duration_seconds`，那么在Prometheus内部衍生3个指标

  - `http_request_duration_seconds_count`: 总请求数
  - `http_request_duration_seconds_sum`: 总耗时
  - `http_request_duration_seconds_bucket`: 边界内桶的计数器

  相关函数

  - `histogram_quantile`

- `Summary`

## Vector

- 范围向量:**某一时间段内**的一组时间序列，每个时间序列包含多个在时间范围内的样本值。**不能直接用于图表展示**，必须是某些函数的输入（如 `rate()`, `irate()`, `avg_over_time()`）。
- 瞬时向量:**某一时刻**的一组时间序列，每个时间序列包含一个样本值
- 范围选择器(`Range Vector Selectors`)

## 面板搭建

多个指标使用同一面板场景

- 同一指标，不同维度

- 相关性指标组合

- 资源消耗全景







### QA

1. query: `sum by(le, path) ( ... )`
   - **`sum()`**：这是一个聚合运算符，它对一组时间序列的值进行求和。
   - **`by(le, path)`**：这是聚合的**分组依据**（Grouping Labels）。它指示 `sum()` 函数：