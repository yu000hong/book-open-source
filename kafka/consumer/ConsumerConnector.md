# ConsumerConnector

> Main interface for consumer.

`ConsumerConnector`是消费者主要的操作接口，要进行Kafka数据消费的话，必须先得生成一个这样的实例。目前只有一个实现类：`ZookeeperConsumerConnector`。

我们看看接口定义哪些方法：

```scala
trait ConsumerConnector {
  
    def createMessageStreams(topicCountMap: Map[String,Int])
        : Map[String, List[KafkaStream[Array[Byte],Array[Byte]]]]
  
    def createMessageStreams[K,V](topicCountMap: Map[String,Int],
                                  keyDecoder: Decoder[K],
                                  valueDecoder: Decoder[V])
        : Map[String,List[KafkaStream[K,V]]]
  
    def createMessageStreamsByFilter[K,V](topicFilter: TopicFilter,
                                          numStreams: Int = 1,
                                          keyDecoder: Decoder[K] = new DefaultDecoder(),
                                          valueDecoder: Decoder[V] = new DefaultDecoder())
        : Seq[KafkaStream[K,V]]

    def commitOffsets(retryOnFailure: Boolean)
  
    def shutdown()
}
```

从接口方法可以看出，我们使用`ConsumerConnector`很简单：

- 使用`createMessageStreams()`或`createMessageStreamsByFilter()`创建KafkaStream进行消费；
- 如果禁用了消费位移的自动提交，那么需要手动调用`commitOffsets()`进行位移提交；
- 完成消费之后退出应用前必须调用`shutdown()`方法释放所有相关资源；

### ZookeeperConsumerConnector

**构造方法**

构造方法里面进行一系列操作，包括：

- 根据配置生成 **consumerIdString** (`${groupId}_${consumerId}`)
- 连接到Zookeeper
- 创建ConsumerFetcherManager
- 建立与OffsetManager的连接（如果位移数据存储到zk就不会执行这一步）
- 如果启用了消费位移的自动提交，那么启动一个线程定时自动提交位移

**createMessageStreams()**

创建消费数据流KafkaStream的过程：

- 根据方法参数构建TopicCount实例
- 将TopicCount数据注册到ZK上，以便所有消费者进行消费平衡计算（分区分配）
- 根据TopicCount获取每个主题对应的ConsumerThreadId列表（逻辑消费列表）
- 根据ConsumerThreadId列表生成对应的消息队列和KafkaStream
- 调用`reinitializeConsumer(topicCount, queuesAndStreams)`方法
- 返回由`ZKRebalancerListener`重平衡计算之后的分区分配结果

**reinitializeConsumer()**

- 创建并注册`ZKRebalancerListener`到ZK
- 创建并注册`ZKSessionExpireListener`到ZK
- 创建并注册`ZKTopicPartitionChangeListener`到ZK
- 调用`syncedRebalance()`主动进行分区分配

