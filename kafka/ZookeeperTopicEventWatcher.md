# ZookeeperTopicEventWatcher


`ZookeeperTopicEventWatcher`两个作用：

- 监听会话状态
- 监听主题列表

### 监听会话状态

如果会话断开之后进行了重连，那么重新注册监听器监听主题列表：`/brokers/topics`。

### 监听主题列表

监听主题列表，也即ZkPath：`/brokers/topics`，获取最新的主题列表。

然后将监听到的最新主题列表交给`TopicEventHandler`处理，目前其只有一个实现`WildcardStreamsHandler`，目的是在主题列表发生变化时，根据我们提供的正则表达式重新进行主题匹配，重新进行消费分配！

### WildcardStreamsHandler

如果主题列表发生了变化，那么会触发我们重新进行匹配计算，得出匹配主题列表。如果新计算出的匹配主题列表与之前的匹配主题列表一致，那么不进行任何操作；如果发现有新增的或减少的主题，那么会调用`reinitializeConsumer(wildcardTopicCount, wildcardQueuesAndStreams)`方法。

注意：只有在我们使用`createMessageStreamsByFilter()`启动消费者的时候才会启用`ZookeeperTopicEventWatcher`监听ZK相关变化来应对主题列表的变动！
