
## ZookeeperConsumerConnector

### consumerIdString

consumerId: 我们可以通过配置`consumer.id`来定义，如果没有定义系统会随机生成一个。

consumerIdString = groupId + "_" + consumerId 

### ensureOffsetManagerConnected()

如果offset存储方式选择的是"kafka"，那么会调用`ConsumerMetadataRequest`请求去任何一个Broker询问consume group对应的协调者Broker，并建立连接：BlockingChannel。

### registerConsumerInZK()

zkPath: `/consumers/$GROUP/ids/$CONSUMER_ID_STRING`

zkData: 

```
{
    "version": 1,
    "subscription": {}, //topic count map
    "pattern": "topic count patter string",
    "timestamp": unix_timestamp_of_now,
}
```

### commitOffsetToZooKeeper()

zkPath: `/consumers/$GROUP/offsets/$TOPIC/$PARTITION_ID`

zkData: `offset`

### fetchOffsetFromZooKeeper()

从`/consumers/$GROUP/offsets/$TOPIC/$PARTITION_ID`获取offset，如果没有对应的offset，那么返回**NoOffset**。

### commitOffsets()

发送`OffsetCommitRequest`请求到协调者Broker

TODO

### fetchOffsets()

如果配置`offsets.storage`的是"zookeeper"，那么从ZK里面获取分区的offsets；如果配置的是"kafka"，那么从协调者Broker请求OffsetFetchRequest，返回分区的offsets。

### reflectPartitionOwnershipDecision()

将重平衡的分配结果写入ZooKeeper（ZK路径：/consumers/$GROUP/owners/$TOPIC/$PARTITION），只要有一个写入失败，那么会回滚删除所有写入结果。

### deletePartitionOwnershipFromZK()

从ZK上删除对应的临时节点（ZK路径：/consumers/$GROUP/owners/$TOPIC/$PARTITION）

### releasePartitionOwnership()

从ZK上删除当前consumer分配的所有分区（ZK路径：/consumers/$GROUP/owners/$TOPIC/$PARTITION）

## TopicCount

消费者在消费时，会指定消费的主题，以及主题对应生成几个Stream，这些数据就是由TopicCount来表示的。

TopicCount的实现类有：
- StaticTopicCount
- WildcardTopicCount

**StaticTopicCount**

StaticTopicCount会指定具体的主题列表，以及每个主题对应生产多少Stream。

**WildcardTopicCount**

WildcardTopicCount通过正则匹配的方式来指定我们具体消费哪些主题，它会去Zookeeper路径`/brokers/topics`下遍历所有主题并进行正则匹配，返回我们需要的主题。

## KafkaStream

一个KafkaStream本质上就是一个逻辑消费线程，不断的从内部BlockingQueue消费数据。而物理拉取线程会从Broker拉取数据填充到这个BlockingQueue里面。

## Zookeeper Structure

### /consumers/$GROUP/ids/$CONSUMER_ID_STRING

data:

```json
{
    "version": 1,
    "subscription": {}, //topic count map
    "pattern": "topic count patter string",
    "timestamp": unix_timestamp_of_now,
}
```

### /brokers/ids/$BROKER_ID

type: **Ephemeral**

data:

```json
{
    "version": 1,
    "host": "broker ip",
    "port": 9092,
    "jmx_port": 8080,
    "timestamp": unix_timestamp_of_now,
}
```

### /brokers/topics/$TOPIC

data:

```json
{
    "partitions": { "$PARTITION_ID": [$BROKER_IDS] }
}
```

### /consumers/$GROUP/owners/$TOPIC/$PARTITION

type: 临时节点

data: ConsumerThreadId

表明分区分配给了某个消费者。


