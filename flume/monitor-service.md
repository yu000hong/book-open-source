# Flume 监控机制

Flume 内部使用JMX MBean的方式来统计各种指标，然后提供了3种方式来暴露指标：

- JMX
- Ganglia
- HTTP

**JMX**

```
-Dcom.sun.management.jmxremote 
-Dcom.sun.management.jmxremote.port=5445 
-Dcom.sun.management.jmxremote.authenticate=false 
-Dcom.sun.management.jmxremote.ssl=false
```

只需要将这几个JAVA选项加到启动命令就可以启动JMX功能了，你就可以使用JConsole来监控指标数据了。你也可以将如下语句加入到`flume-env.sh`文件中，同样也可以启用JMX功能：

```bash
export JAVA_OPTS="-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.port=5445 -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false"
```

**Ganglia**

```bash
bin/flume-ng agent --conf-file example.conf --name a1 -Dflume.monitoring.type=ganglia -Dflume.monitoring.hosts=com.example:1234,com.example2:5455 -Dflume.monitoring.pollFrequency=60 -Dflume.monitoring.isGanglia3=false
```

通过`-Dflume.monitoring.type=ganglia`即启用Ganglia功能，Flume会定期将统计指标上报到Ganglia服务器。`pollFrequency`是可选参数，默认值60秒；`isGanglia3`是可选参数，默认值为false，即使用Ganglia 3.1格式。

**HTTP**

```bash
bin/flume-ng agent --conf-file example.conf --name a1 -Dflume.monitoring.type=http -Dflume.monitoring.port=34545
```

通过`-Dflume.monitoring.type=http`即启用HTTP功能，Flume会启动一个Jetty Server，将统计指标以JSON格式暴露到 **http://\<hostname>:\<port>/metrics** 路径。

### MonitoredCounterGroup

各种组件需要继承`MonitoredCounterGroup`抽象类，统计指标注册到JMX的过程已经由`MonitoredCounterGroup`处理了，因此只需要将统计指标以 **getter** 方法暴露出来即可。

> **⚠️注意:** 目前只支持long型统计数据指标！我们自己开发的组件也可以通过继承它来统计其他更多的指标！

目前已有的实现类有：

- SinkCounter
- KafkaSinkCounter
- SinkProcessorCounter
- ChannelCounter
- FileChannelCounter
- KafkaChannelCounter
- SourceCounter
- KafkaSourceCounter
- ChannelProcessorCounter

### JMXPollUtil

这个类的作用是将Flume里面注册的所有组件统计指标筛选出来，那么它是怎么筛选出来的呢？答案就是通过命名前缀 **org.apache.flume** 。`MonitoredCounterGroup`会将其所有的实现子类以如下的命名方式注册到JMX：`org.apache.flume.$TYPE:type=$NAME`。

其中，`$NAME`为每个组件的名称，`$TYPE`为组件类型。

组件类型有：

- source
- channel_processor
- channel
- sink_processor
- sink
- interceptor
- serializer
- other

`HTTPMetricsServer`和`GangliaServer`都是通过这个类将所有指标从JMX提取出来，然后分别暴露到HTTP端点和上报到Ganglia服务器的。

### 自定义监控服务

除了系统提供的这三种监控方式以外，我们还可以实现自己的监控机制，比如暴露Prometheus端点、比如将数据直接写入InfluxDB/OpenTSDB时序数据库等等。

自定义监控服务只需要如下四步：

- 实现`MonitorService`接口
- 通过`JMXPollUtil`提取统计数据
- 实际工作（暴露统计数据到Prometheus、写入InfluxDB）
- 使用`-Dflume.monitoring.type=YourMonitorService`启用监控服务

> **⚠️注意:** 启动命令参数里面所有以 **flume.monitoring.** 开头的参数都将通过`configure(Context context)`方法被传递到你的自定义MonitorService里面，并且Context里面真正的键会去掉 **flume.monitor.** 前缀。比如"-Dflume.monitoring.zkpath=/flume/monitor"，Context里面将会存在"zkpath=/flume/monitor"这个键值对。

具体实现案例可以参考：[PrometheusMetric](https://github.com/yu000hong/flume-utils/blob/master/src/main/java/com/yu000hong/flume/PrometheusMetric.java)

### 指标解读

首先，我们先来了解几个概念。

`Event`代表一条消息事件，**Source组件**从外部数据源receive消息事件，**Source组件**接受(accept)之后消息事件被put到**Channel组件**，**Sink组件**从Channel中take出消息事件，然后drain到最终目的地。

----

| **Source指标** | Avro | Thrift | Exec | HTTP | JMS | Scribe | Taildir | SpoolDirectory | SyslogTcp | SyslogUDP |
| --- | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| AppendReceivedCount       | ✔ | ✔ |   |   |   |   |   |   |   |   |
| AppendAcceptedCount       | ✔ | ✔ |   |   |   |   |   |   |   |   |
| AppendBatchReceivedCount  | ✔ | ✔ |   | ✔ | ✔ |   | ✔ | ✔ |   |   |
| AppendBatchAcceptedCount  | ✔ | ✔ |   | ✔ | ✔ |   | ✔ | ✔ |   |   |
| ChannelWriteFail          | ✔ | ✔ |   | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| EventReadFail             |   |   |   | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| EventReceivedCount        | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| EventAcceptedCount        | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| OpenConnectionCount       | ✔ |   |   |   |   |   |   |   |   |   |
| GenericProcessingFail     |   |   |   | ✔ |   |   | ✔ | ✔ |   |   |

Avro/Thrift数据源有两种模式：单条(append)和批量(appendBatch)。

单条模式下，每次接收到一条消息事件会自增`AppendReceivedCount`和`EventReceivedCount`指标，当事件被ChannelProcessor成功处理之后(事件被写入Channel)，会自增`AppendAcceptedCount`和`EventAcceptedCount`指标；如果ChannelProcessor处理失败，那么会自增`ChannelWriteFail`指标。

批量模式下，每次接收到一批消息事件会自增`AppendBatchReceivedCount`指标，同时根据消息事件的数量增加`EventReceivedCount`指标，当事件被ChannelProcessor成功处理之后(事件被写入Channel)，会自增`AppendBatchAcceptedCount`指标，同时根据消息事件的数量增加`EventAcceptedCount`指标；如果ChannelProcessor处理失败，那么会自增`ChannelWriteFail`指标。

HTTP/JMS/Scribe/Taildir等从外部数据源读取消息事件如果发生异常，那么会自增`EventReadFail`指标。HTTP/Taildir/SpoolDirectory在将事件写入Channel如果发生**ChannelException**异常，那么会自增`ChannelWriteFail`指标；如果发生其他类型的异常，将会自增`GenericProcessingFail`指标。

----

| **Channel指标** | File | Memory | SpillableMemory | PseudoTxnMemory |
| --- | :---: | :---: | :---: | :---: |
| ChannelCapacity                 | ✔ | ✔ | ✔ |   |
| ChannelSize                     | ✔ | ✔ | ✔ | ✔ |
| CheckpointBackupWriteErrorCount | ✔ |   |   |   |
| CheckpointWriteErrorCount       | ✔ |   |   |   |
| EventPutAttemptCount            | ✔ | ✔ | ✔ | ✔ |
| EventPutErrorCount              | ✔ |   |   |   |
| EventPutSuccessCount            | ✔ | ✔ | ✔ | ✔ |
| EventTakeAttemptCount           | ✔ | ✔ | ✔ | ✔ |
| EventTakeErrorCount             | ✔ |   |   |   |
| EventTakeSuccessCount           | ✔ | ✔ | ✔ | ✔ |

`ChannelCapacity`指的是Channel的最大容量，`ChannelSize`是指Channel当前队列中堆积的事件数量。put表示往Channel中写入事件，take表示从Channel中取出事件，attempt = error + success。

----

| **Sink指标** | Avro/Thrift/HDFSEvent/Hive/HBase2 | AsyncHBase/HBase/ElasticSearch |
| --- | :---: | :---: |
| BatchCompleteCount     | ✔ | ✔ |
| BatchEmptyCount        | ✔ | ✔ |
| BatchUnderflowCount    | ✔ | ✔ |
| ChannelReadFail        | ✔ |   |
| EventWriteFail         | ✔ |   |
| ConnectionClosedCount  | ✔ | ✔ |
| ConnectionCreatedCount | ✔ | ✔ |
| ConnectionFailedCount  | ✔ | ✔ |
| EventDrainAttemptCount | ✔ | ✔ |
| EventDrainSuccessCount | ✔ | ✔ |

每个Sink都会设置一个**BatchSize**属性，每次从Channel中去取消息事件都会尽量多地取事件列表，直到达到BatchSize条或者Channel事件队列被取空。如果取出的事件数量等于BatchSize条，那么自增`BatchCompleteCount`指标；如果取出事件数量为0，那么自增`BatchEmptyCount`指标；如果取出事件数量小于BatchSize条，那么自增`BatchUnderflowCount`指标。

如果从Channel中取消息事件抛出了**ChannelException**异常，那么自增`ChannelReadFail`指标；如果在往外部最终目的地写消息事件抛出异常，那么自增`ChannelReadFail`指标。

从Channel中取出事件列表之后，先根据事件数量增加`EventDrainAttemptCount`指标，成功之后再增加同等数量的`EventDrainSuccessCount`指标。

Sink在与最终目的地建立连接之后，会自增`ConnectionCreatedCount`指标，如果发生异常那么会自增`ConnectionFailedCount`指标；Sink在关闭与最终目的地的连接之后，会自增`ConnectionClosedCount`指标，如果发生一场那么会自增`ConnectionFailedCount`指标。