# Metrics

[Prometheus](https://prometheus.io/)

[OpenMetrics](https://openmetrics.io/)

[Micrometer](https://micrometer.io/)

[Yammer Metrics](https://github.com/dropwizard/metrics/tree/v2.2.0)

[Dropwizard Metrics](https://github.com/dropwizard/metrics)


### Prometheus

Power your metrics and alerting with a leading open-source monitoring solution.

Prometheus is an open-source systems monitoring and alerting toolkit originally built at SoundCloud. 

我们使用Prometheus来存储和处理所有收集到的指标数据，因此后面研究的更多的是：如何将各个不同的指标系统数据导入/接入到Prometheus。本质就是如何写自定义Collector，将各个指标系统的数据适配到simpleclient的Collector，通过collect()方法进行收集并暴露给Prometheus。

### OpenMetrics

OpenMetrics specifies the de-facto standard for transmitting cloud-native metrics at scale, with support for both text representation and Protocol Buffers.

### Micrometer

Micrometer is a metrics instrumentation library for JVM-based applications. It provides a simple facade over the instrumentation clients for the most popular monitoring systems, allowing you to instrument your JVM-based application code without vendor lock-in. It is designed to add little to no overhead to your metrics collection activity while maximizing the portability of your metrics effort.

[Micrometer: Spring Boot 2's new application metrics collector](https://spring.io/blog/2018/03/16/micrometer-spring-boot-2-s-new-application-metrics-collector)

### Yammer Metrics

因为`Kafka-0.8.2`用到了 Yammer Metrics，所以看了下其源代码。Yammer Metrics 是 Dropwizard Metrics 的前身，我们一般使用的版本是：`2.2.0`。Yammer Metrics 的包名是：`com.yammer.metrics`，后面更名为 Dropwizard Metrics 之后，包名也相应发生了改变：`com.codahale.metrics`。

### Dropwizard Metrics

[https://metrics.dropwizard.io/4.1.2/](https://metrics.dropwizard.io/4.1.2/)

Dropwizard Metrics is a Java library which gives you unparalleled insight into what your code does in production. It provides a powerful toolkit of ways to measure the behavior of critical components in your production environment.

[Spring Boot 2: Migrating From Dropwizard Metrics to Micrometer](https://dzone.com/articles/spring-boot-2-migrating-from-dropwizard-to-micrometer)

Spring Boot 2 已经从 Dropwizard Metrics 迁移到了 Micrometer，所以我们也尽量使用 Micrometer 吧。

