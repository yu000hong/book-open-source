# TopicCount

我们先看看`TopicCount`定义：

```scala
private[kafka] trait TopicCount {
  def getConsumerThreadIdsPerTopic: Map[String, Set[ConsumerThreadId]]
  def getTopicCountMap: Map[String, Int]
  def pattern: String
}
```

主要定义了三个方法：

- `pattern()`：返回当前匹配模式，有**static** / **white_list** / **black_list**三种值；
- `getTopicCountMap()`：返回主题对应的数据流Consumer的数量；
- `getConsumerThreadIdsPerTopic`：返回主题对应的所有**ConsumerThreadId**列表；

`TopicCount`有两个实现类：

- StaticTopicCount
- WildcardTopicCount

### StaticTopicCount

为我们要消费的每个主题指定固定的数据流Consumer数量，不同主题可以指定不同的数据流Consumer数量。

### WildcardTopicCount

我们只需要提供一个正则表达式，它会自动连接ZK，然后从`/brokers/topics`目录下去匹配所有我们需要消费的主题。这种方式下，所有主题只能指定相同数量的数据流Consumer。

根据`TopicFilter`的不同实现，有两种模式：
- **white_list**: TopicFilter.Whitelist
- **black_list**: TopicFilter.Blacklist

### constructTopicCount()

方法**constructTopicCount()**的作用是从Zookeeper读取消费者的消费分区配置，用于后面的分区分配计算。

每个消费者的`TopicCount`数据都会写入到ZK的`/consumers/$GROUP/ids/$CONSUMER_ID_STRING`路径下，具体写入内容为：

```
{
    "version": 1,
    "subscription": {}, //topic count map
    "pattern": "static|white_list|black_list",
    "timestamp": unix_timestamp_of_now,
}
```

其中，**subscription**数据就是消费者指定的`TopicCount`数据，如：

```
{
    "version": 1,
    "subscription": {
        "follow": 10,
        "click": 15,
        "view": 18,
    }, 
    "pattern": "static",
    "timestamp": unix_timestamp_of_now,
}
```

```
{
    "version": 1,
    "subscription": {
        "user_*": 10,
    }, 
    "pattern": "white_list|black_list",
    "timestamp": unix_timestamp_of_now,
}
```

为什么这些数据要写入Zookeeper呢？为了达到消费均衡，所有的消费者都会将自己需要的主题以及消费能力(数据流个数)都写入ZK，然后每个消费者都会去ZK上读取这些数据，再根据这些数据进行**相同的计算**，最后达成一致决议。

这里`相同的计算`很重要，如果大家计算方式不一样，会导致形成的决议不一致，导致消费出现问题。比如某个主题有10个分区，3个消费者，其中AB两个消费者计算结果是：A消费分区1-4，B消费分区5-7，C消费分区8-10。但是C消费者由于配置不一致，计算结果是：A消费分区1、4、7、10，B消费分区2、5、8，C消费分区3、6、9。

最后的消费结果就是：A消费分区1-4，B消费分区5-7，C消费分区3、6、9。分区3和6重复消费了，但是分区8和10却没有被消费。出现这种情况的原因就是AB消费者配置了`RangeAssignor`，而C配置了`RoundRobinAssignor`两种不一致的分区分配计算方式！




