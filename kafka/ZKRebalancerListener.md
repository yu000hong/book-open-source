# ZKRebalancerListener

### 主要字段

- partitionAssignor：分区分配器
- isWatcherTriggered：是否触发再平衡

### 再平衡线程

有一个专门的线程用于对分区消费任务进行再平衡，这个线程会根据`isWatcherTriggered`的值来决定是否需要进行再平衡计算。如果没有触发再平衡，那么等待一段时间；如果触发了再平衡，那么调用`syncedRebalance()`进行重新分配。

那么什么时候会触发再平衡计算呢？

有三个时机会触发再平衡计算过程：

1. 当ZK监听器监听到 **/brokers/topics/$TOPIC** 结点内容即主题元数据发生变化时
2. 当ZK监听器监听到 **/consumers/$GROUP/ids** 目录子结点即消费者发生变化时
3. 当ZK监听器监听到 **/brokers/ids** 目录子结点即集群Broker发生变化时（极端情况）


### 主要方法

- rebalanceEventTriggered()
- syncedRebalance()
- rebalance()
- reflectPartitionOwnershipDecision()
- deletePartitionOwnershipFromZK()
- releasePartitionOwnership()
- addPartitionTopicInfo()
- resetState()
- closeFetchersForQueues()
- clearFetcherQueues()
- closeFetchers()
- updateFetcher()

### rebalanceEventTriggered()

设置`isWatcherTriggered`字段为 **true**，唤醒再平衡线程，触发其进行再平衡计算。

### rebalance() & syncedRebalance()

`rebalance()`里面进行了再平衡计算，`syncedRebalance()`采用了加锁的方式调用`rebalance()`进行再平衡计算。他们再平衡计算逻辑都是在`rebalance()`方法里面，他们的区别有两个：

- 是否加锁：外部使用也只使用了加锁版本`syncedRebalance()`，未加锁版本`rebalance()`只是作为一个私有方法；
- 重试逻辑：`syncedRebalance()`含有重试逻辑，如果超过配置的最大重试次数仍然报错，那么消费者会直接抛出异常；如果是在消费者启动过程中报错，那么会导致消费者启动失败；如果是在再平衡线程中报错，再平衡线程会记录失败日志。（TODO：有其他影响么？）


**再平衡过程**

- 调用`closeFetchers()`停止所有拉取线程，同时清除队列里面的所有数据（如果开启了消费位移自动提交，那么会提交消费位移；如果手动提交的话，在再平衡过程中，可能会导致消息数据被重复消费）；
- 调用`releasePartitionOwnership()`方法删除ZK上对应的所有临时结点；
- 使用`PartitionAssignor`进行分区分配计算，算出当前消费者需要消费的分区列表；
- 调用`fetchOffsets()`去 **OffsetManager** 获取所有分区的消费位移；
- 调用`addPartitionTopicInfo()`重新设置 **PartitionTopicInfo**；
- 调用`reflectPartitionOwnershipDecision()`将当前消费者分配的分区列表反映到ZK上；
- 调用`updateFetcher()`告知 **ConsumerFetcherManager** 开启所有拉取线程；

### reflectPartitionOwnershipDecision()

当再平衡计算完成之后，会得到一个分区任务分配结果，这个结果最终要写入到ZK里去。

ZkPath：**/consumers/$GROUP/owners/$TOPIC/$PARTITION**

ZkData：**$CONSUMER_THREAD_ID**

整个写入过程是保证原子性的，如果有任何一个结点写入失败，那么会删除所有已经写入的临时结点，同时返回 **false**；否则，返回 **true**。

什么情况下会写入失败呢？写入失败的话会导致什么样的结果呢？

> 如果我们同一个组的消费者配置了不同的分区任务分配策略，那么可能导致消费者各自计算出来的分配结果不一致。某个分区被多个消费者消费，或者某个分区可能没有消费者去消费。如果某个分区被多个消费者消费了，那么会导致后去写入分配结果的消费者没法成功写入ZK，因为分区对应的临时结点已经存在了。会导致后面这个消费者的再平衡计算过程失败，抛出异常`ConsumerRebalanceFailedException`，这个异常一直会抛到最上层，导致消费者失败，甚至导致整个应用崩溃。
>
> 这样的话，配置异常的消费者就会直接挂掉，剩下的消费者重新进行再平衡计算瓜分分区任务，如果没有出现上述错误情况的话，那么剩下的消费者将会正常的进行分区消费了。

（TODO）补充问题：由于同一个组内的消费者配置了不同的分区任务分配策略，会不会出现没有重复消费分区的情况，但仅仅出现某个分区没有被消费到的情况呢？

> 就我个人目前的感觉，应该不会出现这种情况，后面再好好研究研究。


### releasePartitionOwnership()

`deletePartitionOwnershipFromZK()`直接从ZK路径 **/consumers/$GROUP/owners/$TOPIC/$PARTITION**删除对应结点。`releasePartitionOwnership()`会删除当前消费者分配的所有分区对应的临时结点。

### clearFetcherQueues()

两个操作：

- 清除BlockingQueue队列里面的所有数据（原始数据FetchedDataChunk）；
- 清除KafkaStream数据流的所有数据（反序列化解码之后的数据）；

### closeFetchersForQueues()

停止所有拉取线程`ConsumerFetcherThread`，同时调用`clearFetcherQueues()`清除队列的所有数据。

如果配置了消费位移自动提交的话，还要调用`commitOffsets()`提交当前的消费位移。

### closeFetchers()

直接调用`closeFetchersForQueues()`关闭所有拉取线程。

### updateFetcher()

根据 **topicRegistry** 里面的分区任务数据，开启所有拉取线程。

### addPartitionTopicInfo()

往 **topicRegistry** 字段里面添加`PartitionTopicInfo`，本质就是给 **Fetcher** 添加需要拉取的分区。后面会通过`updateFetcher()`方法来启动所有拉取线程。

### resetState()

`resetState()`方法直接清理掉了所有分区相关的数据，包括：

- 拉取位移
- 消费位移
- 消息队列

```scala
def resetState() {
    topicRegistry.clear
}
```
