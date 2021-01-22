# Yammer Metrics

[Github](https://github.com/dropwizard/metrics/tree/v2.2.0)

`Yammer Metrics`是`Dropwizard Metrics`的前身，因为Kafka-0.8.2里用到了它，所以这里简单看了下源码，方便将其适配到Prometheus simpleclient的Collector。

### 指标类型

- Counter
- Gauge
    - ToggleGauge
    - JmxGauge
    - RatioGauge
    - PercentGauge
- Histogram
- Meter
- Timer

### Counter

Yammer Metrics中的Counter可增可减，而Prometheus中的Counter只能增加，所以这个Counter很难适配到Prometheus中去，不过可以适配到Gauge，但其实语义其实不太适合。

### Gauge

Yammer Metrics中的Gauge<T>是泛型化的，它可以存储任意值，与Prometheus中的Gauge不太匹配。Prometheus中的Guage只处理数字类型，所以数字类型的Guage可以适配，其他类型就没办法处理了。

- ToggleGauge：只返回0或者1
- JmxGauge：从JMX中获取值
- RatioGauge：返回比率值
- PercentGauge：返回百分比

### Histogram

提供了如下几个数据：

- count
- sum
- max
- min
- mean
- stdDev
- quantile: 0.5/0.75/0.95/0.98/0.99/0.999

Yammer Metrics 中的 Histogram 可以直接映射为 Prometheus 中的 Summary 类型。

### Meter

提供如下几个数据：

- count
- oneMinuteRate
- fiveMinuteRate
- fifteenMinuteRate
- meanRate

Yammer Metrics 中的 Meter 提供了5种数据，与 Prometheus 中的指标类型映射关系如下：

- count: Counter
- oneMinuteRate: Gauge
- fiveMinuteRate: Gauge
- fifteenMinuteRate: Gauge
- meanRate: Gauge

### Timer

Yammer Metrics 中的 Timer 底层实际上是由一个 Meter 和一个 Histogram 组成，提供如下指标数据：

- count
- sum
- max
- min
- mean
- stdDev
- quantile: 0.5/0.75/0.95/0.98/0.99/0.999

- oneMinuteRate: Gauge
- fiveMinuteRate: Gauge
- fifteenMinuteRate: Gauge
- meanRate: Gauge

因此，可以映射到 Prometheus 中到一个 Summary 和几个 Gauge 类型。
