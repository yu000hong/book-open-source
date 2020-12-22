# ChannelProcessor

ChannelProcessor的作用就是将Source接收到的消息事件写入(Put)到Channel，两个核心方法就是：

- `void processEventBatch(List<Event> events)`
- `void processEvent(Event event)`

### ChannelSelector

官方文档：[Flume Channel Selectors](https://flume.apache.org/releases/content/1.9.0/FlumeUserGuide.html#flume-channel-selectors)

每个ChannelProcessor都会有一个对应的ChannelSelector，用于选择哪些Channel是必须的，哪些是可选的。我们看看ChannelSelector的定义就清楚了：

```java
interface ChannelSelector extends NamedComponent, Configurable {
    void setChannels(List<Channel> channels);
    List<Channel> getRequiredChannels(Event event);
    List<Channel> getOptionalChannels(Event event);
    List<Channel> getAllChannels();
}
```

- 对于 **RequiredChannels** ，只要其中一个写入抛出异常，最后都会抛出`ChannelException`（除非ERROR错误）；对于 **OptionalChannels**，只要不是ERROR类型错误，其他异常都会被直接吞掉。
- **RequiredChannels** 必须至少配置一个Channel，**OptionalChannels** 可以不用配置。

`ChannelSelector` 的实现类有两个：

- ReplicatingChannelSelector: 将消息事件同时写入到配置的所有Channel里
- MultiplexingChannelSelector: 将消息事件根据Header值分发写入到不同的Channel里

**ReplicatingChannelSelector**

```ini
a1.sources = r1
a1.channels = c1 c2 c3
a1.sources.r1.channels = c1 c2 c3
a1.sources.r1.selector.type = replicating
a1.sources.r1.selector.optional = c3
```

默认的type就是 **replicating**，因此可以不用显示配置type值。从配置中我们可以看出，`r1`配置了三个Channel：c1, c2, c3。c3为optional，那么其余两个c1和c2即为required。

**MultiplexingChannelSelector**

```ini
a1.sources = r1
a1.channels = c1 c2 c3 c4 c5 c6 c7
a1.sources.r1.channels = c1 c2 c3 c4 c5 c6 c7
a1.sources.r1.selector.type = multiplexing
a1.sources.r1.selector.header = state
a1.sources.r1.selector.mapping.CZ = c1
a1.sources.r1.selector.mapping.US = c2 c3
a1.sources.r1.selector.optional.CZ = c4 c5
a1.sources.r1.selector.optional.US = c6
a1.sources.r1.selector.default = c7
```

`MultiplexingChannelSelector` 将根据我们配置的Header值来进行分发，`selector.mapping.`配置的是 **RequiredChannels**，`selector.optional.`配置的是 **OptionalChannels**。

如果我们在处理 **RequiredChannels** 时，根据Header值没有找到对应的Channels，那么将会使用`selector.default`配置的默认Channels；如果我们在处理 **OptionalChannels** 时，根据Header值没有找到对应的Channels，那么不会有任何写入，直接忽略；

> **注意:** 无论我们配置的是`MultiplexingChannelSelector`还是`ReplicatingChannelSelector`，**RequiredChannels** 和 **OptionalChannels** 都必须同时处理，也即是都要走写入Channels逻辑，**RequiredChannels** 写入成功之后，还必须要写入 **OptionalChannels** ！