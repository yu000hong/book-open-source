# 消费位移

`ZookeeperConsumerConnector`中与消费位移有关的方法有：

- `fetchOffsetFromZooKeeper()`
- `fetchOffsets()`
- `commitOffsetToZooKeeper()`
- `commitOffsets()`
- `autoCommit()`

### fetchOffsetFromZooKeeper()

方法定义：

```scala
def fetchOffsetFromZooKeeper(topicPartition: TopicAndPartition)
```

从ZK上读取当前组在某个主题的某个分区的消费位置。

ZkPath：`/consumers/$GROUP/offsets/$TOPIC/$PARTITION`

ZkData：`$OFFSET` (消费位移整数)

### fetchOffsets()

方法定义：

```scala
def fetchOffsets(partitions: Seq[TopicAndPartition])
```

从ZK或者Broker上(根据配置决定)读取当前组在某些主题对应的分区下的消费位移。从ZK上读取消费位移的功能最后是委托给上述的`fetchOffsetFromZooKeeper()`方法，通过多次调用获取所有分区对应的消费位移。这里我们着重讲一下从Broker获取消费位移的过程：

- 第一步：连接到消费位移对应的Broker
- 第二步：给Broker发送获取消费位移的请求
- 第三步：如果设置了双重提交，那么再与ZK里面的消费位移比较，选择最大位移

**第一步：连接到消费位移对应的Broker**

首先我们从ZK对应路径`/brokers/ids`下找出所有的Brokers；

然后随机连接一个Broker，发送`ConsumerMetadataRequest`请求，也就是当前组的消费位移存储在哪个Broker；

如果选择的Broker存储了当前组的消费位移，那么就会返回正确的`ConsumerMetadataResponse`；否则随机选择下一个Broker再进行相同的请求，直到找到对应的存储消费位移的Broker；

**第二步：给Broker发送获取消费位移的请求**

给选定的Broker发送`OffsetFetchRequest`，然后从Broker返回`OffsetFetchResponse`。这个过程中，有可能存储消费位置的分区Leader发生了改变，这时候会重新回到第一步连接新的 **Leader Broker** 去获取消费位移。

**第三步：如果设置了双重提交，那么再与ZK里面的消费位移比较，选择最大位移**

设置双重提交的目的是为了将消费位移从ZK迁移到Broker，如果没有设置双重提交，那么就直接返回消费位移；如果设置了双重提交，就会比较ZK和Broker里面存储的位移，选择最大位移返回。

### commitOffsetToZooKeeper()

将消费位移写入到ZK上。

ZkPath：`/consumers/$GROUP/offsets/$TOPIC/$PARTITION`

ZkData：`$OFFSET` (消费位移整数)

### commitOffsets()

**topicRegistry** 字段里面每个分区元数据`PartitionTopicInfo`中的 **consumedOffset** 记录了我们当前消费者对该分区的消费位移，调用`commitOffsets()`会将这些消费位移提交到ZK或者OffsetManager。如果配置的是OffsetManager，同时配置了双重提交，那么再提交到OffsetManager成功之后，还会将消费位移提交到ZK。

### autoCommit()

`autoCommit()`方法内部就是直接调用的`commitOffsets(false)`来完成消费位移提交的，这个方法唯一的使用场景就是当消费者开启自动提交的时候，会启动一个线程定期的调用`autoCommit()`方法完成消费位移提交。

