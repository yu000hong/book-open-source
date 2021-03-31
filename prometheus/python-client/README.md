# prometheus-client

### prometheus_multiproc_dir

`prometheus_multiproc_dir`这个参数定义了我们的环境为多进程环境，还是单进程环境。

看看代码：

**values.py**

```python
def get_value_class():
    # Should we enable multi-process mode?
    # This needs to be chosen before the first metric is constructed,
    # and as that may be in some arbitrary library the user/admin has
    # no control over we use an environment variable.
    if 'prometheus_multiproc_dir' in os.environ:
        return MultiProcessValue()
    else:
        return MutexValue
```

### 一些概念

**Collector**

> Collectors must have a no-argument method 'collect()' that returns a list of `Metric` objects. The returned metrics should be consistent with the Prometheus exposition formats.

**CollectorRegistry**

> 指标收集器注册的地方，我们一般使用一个CollectorRegistry就够了，然后在Prometheus来爬取(scape)指标数据时，使用`collectorRegistry.collect()`方法将所有收集器收集到的指标暴露出去给Prometheus。一般我们无需配置，直接使用的默认注册器：
> `REGISTRY = CollectorRegistry(auto_describe=True)`。
>
> 当然，我们也可以使用多个注册器进行分组收集指标。那样的话，我们需要在定义指标的时候指定我们的分组，也就是具体注册到哪个注册器：

```python
myRegistry = CollectorRegistry(auto_describe=True)
myCounter = Counter("request_total", "count all the requests", registry=myRegistry)
myCounter.inc()
```

**MetricWrapperBase**

指标包装基类，它本质上也是收集器Collector，具有不带参数的`collect()`方法。子类有：

- Counter
- Gauge
- Summary
- Histogram
- Info
- Enum

指标包装类型有模版父类的概念，模版父类定义了指标具有的所有Tag名称，然后有具体的指标类来填充Tag值。如：

```python
c = Counter('my_requests_total', 'HTTP Failures', ['method', 'endpoint'])
c.labels('get', '/').inc() # 这里会按照模板生成一个全新的Counter，或者从缓存中获取已有的Counter
c.labels('post', '/submit').inc()
```

> **注意⚠️**：上面代码中我们无法直接使用`c.inc()`，因为c是一个模版父类，没有指定指标Tag具体值；只有在指定了指标Tag具体值之后，它才算一个真正可操作的指标。每次调用`labels(..)`方法都会从缓存中查找对应指标，或按照模版父类生成一个全新的指标并缓存起来。
>
> 所有的子指标都会全部缓存起来，所以，如果Tag具有很多不同具体值时，会耗掉较多的内存，甚至OOM！

**Metric**

> A single metric family and its samples.

`Metric`代表了指标对应的一组具有同样Tag列表的采样值，它是基类，子类有：

- CounterMetricFamily
- GaugeMetricFamily
- SummaryMetricFamily
- HistogramMetricFamily
- GaugeHistogramMetricFamily
- InfoMetricFamily
- StateSetMetricFamily
- UnknownMetricFamily/UntypedMetricFamily

主要字段：

- name: 指标名称
- documentation: 指标描述
- type: 指标类型
- unit: TODO
- samples: 指标采样数据数组
- _labelnames: Tag名称列表

**Sample**

采样数据，类型为命名元组。

```python
Sample = namedtuple('Sample', ['name', 'labels', 'value', 'timestamp', 'exemplar'])
```

**labels**是一个dict类型，key为Tag名，value为Tag值。




