# 消费者启动过程

- 生成ConsumerIdString
- 创建ConsumerFetcherManager
- 连接到Zookeeper
- 建立与OffsetManager Broker的连接（如果位移数据存储到zk就不会执行这一步）
- 如果启用了自动提交位移(auto.commit.enable=true)，那么启动一个线程定时自动提交

用户调用如下三个方法之一开启消费过程：

- def createMessageStreams(topicCountMap: Map[String,Int])
- def createMessageStreams[K,V](topicCountMap: Map[String,Int], keyDecoder: Decoder[K], valueDecoder: Decoder[V])
- def createMessageStreamsByFilter[K,V](topicFilter: TopicFilter,
                                        numStreams: Int,
                                        keyDecoder: Decoder[K] = new DefaultDecoder(),
                                        valueDecoder: Decoder[V] = new DefaultDecoder())

> **注意⚠️**：这三个方法只能调用其中一个，不能同时调用。调用任意一个方法就确定了这个消费者需要消费哪些主题，每个主题有几个"线程"(数据流KafkaStream)，确定之后就不能再做修改！

**createMessageStreams**

指定具体的主题，以及主题需要生成的数据流个数

**createMessageStreamsByFilter**

可以通过正则表达式来匹配我们需要消费的主题，每个主题生成numSterams个数据流。注意使用正则匹配方式的话，匹配出来的每个主题都只能生成相同数量的数据流。

### 消费启动过程

- 根据createMessageStreams方法提供的TopicCount，建立主题到消费者逻辑线程集合的映射consumerThreadIdsPerTopic；
- 根据逻辑线程生成对应的LinkedBlockingQueue[FetchedDataChunk]和KafkaStream；Fetcher线程从Broker拉取的消息数据FetchedDataChunk之后，会放入这个队列，用户直接使用的是KafkaStream这个对象，而KafkaStream本质上就是对这个队列的一些封装；
- 将消费者注册到ZK的`/consumers/$GROUP/ids/$CONSUMER_ID_STRING`这个路径下，同时写入数据：

    ```json
    {
        "version": 1,
        "subscription": {}, //topic count map
        "pattern": "topic count patter string",
        "timestamp": unix_timestamp_of_now,
    }
    ```

- 启动分区消费负载均衡器ZKRebalancerListener，ZKRebalancerListener会监听`/consumers/$GROUP/ids`路径，如果发生变化将启动重平衡过程；
- 启动ZKSessionExpireListener，监听会话状态的变更；如果会话状态变更的话，将会重置ZKRebalancerListener状态，重新注册消费者到ZK，同时启动重平衡过程；
- 启动ZKTopicPartitionChangeListener，监听消费者消费的所有主题`/brokers/topics/$TOPIC`，如果监听到数据变化（也即是分区变化），将会启动重平衡过程；
- 启动重平衡过程；
