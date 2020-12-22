
### Source 和 Sink 共同点

Sink只有一种类型，采用轮询机制去轮询Channel。

Source有两种类型：

- PollableSource：轮询机制轮询外部数据源
- EventDrivenSource：事件驱动机制

轮询机制实现有：

- KafkaSource
- JMSSource
- TaildirSource

事件驱动机制实现有：

- NetcatSource
- HTTPSource
- ScribeSource
- AvroSource
- ThriftSource
- EmbeddedSource
- ExecSource
- SpoolingDirectorySource


PollableSource 和 Sink 都是采用的轮询机制，它们有一个共同的方法：

```java
Status process() throws EventDeliveryException;
```

Status是一个枚举类，有两个值：READY 和 BACKOFF。

返回READY表示还有数据需要继续处理；返回BACKOFF表示没有数据，需要进行退避休眠。

同时，它们还有同样的退避休眠机制，有两个配置参数：

- backoffSleepIncrement
- maxBackoffSleep

不一样的是，这两个配置一个是在Source里面配置，一个是在SinkRunner里面配置！



