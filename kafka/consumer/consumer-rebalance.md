# 消费者重平衡


### PartitionAssignor

Assigns partitions to consumer instances in a group.

**AssignmentContext**

AssignmentContext是指定的某个Consumer对应的上下文环境，有四个字段：

- myTopicThreadIds: consumer下的主题和当前consumer对应的ConsumerThreadId列表的映射
- partitionsForTopic: consumer下的主题和所有分区ID列表的映射
- consumersForTopic: 消费者组下主题对应的所有ConsumerThreadId列表的映射
- consumers: 消费者组下的所有consumer

这些数据都是从ZK上获取的，myTopicThreadIds是从`/consumers/$GROUP/ids/$CONSUMER_ID_STRING`节点的数据获取的，partitionsForTopic是从`/brokers/topics/$TOPIC`节点的data数据获取的，consumersForTopic是遍历`/consumers/$GROUP/ids`下所有子节点及其data数据获取的，consumers是遍历`/consumers/$GROUP/ids`下所有子节点获取的。

**RoundRobinAssignor**

RoundRobinAssignor要求同一个消费者组下的所有消费者必须订阅相同的主题列表，否则会抛异常！

分配方式：

- 分配是以所有主题打乱进行的
- 将主题及其分区构建成TopicAndPartition对象
- 将所有TopicAndPartition按照其哈希码进行排序
- 将所有ConsumerThreadId进行排序，排成一个圈
- 将TopicAndPartition依次分配给ConsumerThreadId

> 如果TopicAndPartition的数量小于ConsumerThreadId的数量，那么将会有部分ConsumerThreadId分配不到，导致其空转（空转ConsumerThreadId好像会被回收）

> 如果TopicAndPartition的数量是ConsumerThreadId数量的整数倍，那么每个ConsumerThreadId将会分配到相同数量的分区；否则某些ConsumerThreadId将会少分配一个分区

**RangeAssignor**

分配方式：

- 分配是以主题为单位的
- 取出主题下的所有分区
- 取出主题下的所有消费者ConsumerThreadId
- 取出主题下的所有分区
- 计算：分区总数/消费者总数 = n
- 计算：分区总数%消费者总数 = m
- 每个消费者按顺序从分区列表里面领取分区
- 排在[0,m)位置的消费者(前m个消费者)领取n+1个分区
- 排在后面位置的消费者领取n个分区

