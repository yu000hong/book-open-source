# Gunicorn启用Prometheus进行指标收集

官方文档：[Multiprocess Mode (Gunicorn)](https://github.com/prometheus/client_python#multiprocess-mode-gunicorn)

> Prometheus client libraries presume a threaded model, where metrics are shared across workers. This doesn't work so well for languages such as Python where it's common to have processes rather than threads to handle large workloads.
>
> To handle this the client library can be put in multiprocess mode. This comes with a number of limitations:
>
> - Registries can not be used as normal, all instantiated metrics are exported
> - Custom collectors do not work (e.g. cpu and memory metrics)
> - Info and Enum metrics do not work
> - The pushgateway cannot be used
> - Gauges cannot use the pid label

Gunicorn启用Prometheus的步骤如下：

- 安装必须依赖
- 配置环境变量
- 暴露指标接口
- 清理指标文件
- 使用指标类型

### 安装必须依赖

```bash
pip install prometheus-client
```

### 配置环境变量

必须配置`prometheus_multiproc_dir`环境变量，才能开启多进程模式。而且，这个环境变量必须在启动脚本里面设置才能多所有子进程生效，不能在程序里面使用Python去设置。

同时，这个目录必须存在，否则启动会报错！

```bash
$ FLASK_APP=api FLASK_ENV=local prometheus_multiproc_dir=metric /usr/local/bin/gunicorn --preload -c gunicorn_local.py api:app
```

### 暴露指标接口

我们必须提供一个接口，让Prometheus定期来抓取数据。

```python
from flask import Blueprint, Response
from prometheus_client import multiprocess, generate_latest, REGISTRY, CONTENT_TYPE_LATEST

metric_bp = Blueprint('metric', __name__, url_prefix='/metric')
@metric_bp.route('', methods=['GET', 'POST'])
def metric():
    multiprocess.MultiProcessCollector(REGISTRY)
    data = generate_latest(REGISTRY)
    return Response(data, mimetype=CONTENT_TYPE_LATEST)
```

### 清理指标文件

由于采用的多进程模式，会在配置的目录下生成内存映射文件，所以在程序退出的时候需要清理这些文件。

```python
def child_exit(server, worker):
    from prometheus_client import multiprocess
    multiprocess.mark_process_dead(worker.pid)
```

`mark_process_dead`只会清理 **guage_livesum** 和 **guage_liveall** 类型。在子进程退出的时候，我们只能清理这两类类型的指标数据，其他类型的数据不能清，清了就会影响统计。比如：清了某个子进程的Counter类型数据，那么这个Counter数据就会突然少掉一部分，逻辑上是不应该少的。

```python
def mark_process_dead(pid, path=None):
    """Do bookkeeping for when one process dies in a multi-process setup."""
    if path is None:
        path = os.environ.get('prometheus_multiproc_dir')
    for f in glob.glob(os.path.join(path, 'gauge_livesum_{0}.db'.format(pid))):
        os.remove(f)
    for f in glob.glob(os.path.join(path, 'gauge_liveall_{0}.db'.format(pid))):
        os.remove(f)
```

在Gunicorn进程退出之后，我们就得清理指定的指标目录下的所有指标文件，清理的时机放在`on_exit()`配置里面。

```python
def on_exit(server):
    path = os.environ.get('prometheus_multiproc_dir')
    for f in glob.glob(os.path.join(path, '*.db')):
        os.remove(f)
```

### 使用指标类型

**Counter**

Counters go up, and reset when the process restarts.

```python
from prometheus_client import Counter
c = Counter('my_failures', 'Description of counter')
c.inc()     # Increment by 1
c.inc(1.6)  # Increment by given value
```

If there is a suffix of **_total** on the metric name, it will be removed. When exposing the time series for counter, a **_total** suffix will be added. This is for compatibility between OpenMetrics and the Prometheus text format, as OpenMetrics requires the **_total** suffix.

There are utilities to count exceptions raised:

```python
@c.count_exceptions()
def f():
  pass

with c.count_exceptions():
  pass

# Count only one type of exception
with c.count_exceptions(ValueError):
  pass
```

**Gauge**

Gauges can go up and down.

```python
from prometheus_client import Gauge
g = Gauge('my_inprogress_requests', 'Description of gauge')
g.inc()      # Increment by 1
g.dec(10)    # Decrement by given value
g.set(4.2)   # Set to a given value
```

There are utilities for common use cases:

```python
g.set_to_current_time()   # Set to current unixtime

# Increment when entered, decrement when exited.
@g.track_inprogress()
def f():
  pass

with g.track_inprogress():
  pass
```

A Gauge can also take its value from a callback:

```python
d = Gauge('data_objects', 'Number of objects')
my_dict = {}
d.set_function(lambda: len(my_dict))
```

**Summary**

Summaries track the size and number of events.

```python
from prometheus_client import Summary
s = Summary('request_latency_seconds', 'Description of summary')
s.observe(4.7)    # Observe 4.7 (seconds in this case)
```

There are utilities for timing code:

```python
@s.time()
def f():
  pass

with s.time():
  pass
```

The Python client doesn't store or expose quantile information at this time.

**Histogram**

Histograms track the size and number of events in buckets. This allows for aggregatable calculation of quantiles.

```python
from prometheus_client import Histogram
h = Histogram('request_latency_seconds', 'Description of histogram')
h.observe(4.7)    # Observe 4.7 (seconds in this case)
```

The default buckets are intended to cover a typical web/rpc request from milliseconds to seconds. They can be overridden by passing buckets keyword argument to Histogram.

There are utilities for timing code:

```python
@h.time()
def f():
  pass

with h.time():
  pass
```
