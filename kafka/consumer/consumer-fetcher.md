# ConsumerFetcherThread & ConsumerFetcherManager

### AbstractFetcherThread

> Abstract class for fetching data from multiple partitions from the same broker.

这个`AbstractFetcherThread`不仅用于Consumer，Kafka内部逻辑也要使用它，所以它位于 **kafka.server** 包内。它继承自`java.lang.Thread`，所以它会启动一个线程去完成分区数据的拉取工作。

两个实现类：

- ConsumerFetcherThread：消费者从Broker拉取消息
- ReplicaFetcherThread：副本从Leader复制拉取消息

三个抽象方法：

- processPartitionData：如何处理从Broker拉取的分区数据
- handleOffsetOutOfRange：当消费位移出现非法位置的时候如何处理
- handlePartitionsWithErrors：怎么处理异常情况，如分区Leader发生改变

```scala
// process fetched data
def processPartitionData(topicAndPartition: TopicAndPartition, fetchOffset: Long,
                        partitionData: FetchResponsePartitionData)

// handle a partition whose offset is out of range and return a new fetch offset
def handleOffsetOutOfRange(topicAndPartition: TopicAndPartition): Long

// deal with partitions with errors, potentially due to leadership changes
def handlePartitionsWithErrors(partitions: Iterable[TopicAndPartition])
```

需要拉取分区的相关方法：

- addPartitions：为Fetcher添加要拉取的分区信息
- removePartitions：从已有的分区列表里移除某些分区
- partitionCount：目前分区列表里需要拉取的分区总数

这三个方法都是针对字段`partitionMap HashMap[TopicAndPartition, Long]`进行操作的方法。

**doWork()**

`doWork()`方法相当于`Thread.run()`方法，承载了线程处理的所有逻辑，我们看看源码：

```scala
override def doWork() {
    inLock(partitionMapLock) {
        //如果分区列表为空，那么等待200毫秒
        if (partitionMap.isEmpty) partitionMapCond.await(200L, TimeUnit.MILLISECONDS)
        partitionMap.foreach {
            case((topicAndPartition, offset)) =>
                //将所有分区添加到FetchRequestBuilder里去
                fetchRequestBuilder.addFetch(topicAndPartition.topic, 
                    topicAndPartition.partition, offset, fetchSize)
        }
    }
    //构建拉取请求
    val fetchRequest = fetchRequestBuilder.build()
    //执行拉取逻辑
    if (!fetchRequest.requestInfo.isEmpty) processFetchRequest(fetchRequest)
}
```

从代码可以看出，重要的逻辑都封装在了`processFetchRequest()`方法里了。

**processFetchRequest()**

最终的拉取请求是委托给`SimpleConsumer`去处理的，具体可以参考：[SimpleConsumer](SimpleConsumer.md)

我们看看这个方法的处理逻辑：

- 通过`SimpleConsumer`发送拉取请求，获取分区数据(FetchResponse)；
- 逐个分区地遍历FetchResponse里返回的分区数据；
- 如果分区对应的响应为 **NoError**，那么写入分区最新位移，同时调用`processPartitionData()`处理分区数据；
- 如果分区对应的响应为 **OffsetOutOfRangeCode**，那么调用`handleOffsetOutOfRange()`方法计算一个新的位移，并放到存储消费位移的`partitionMap`字段里，供下一次拉取时进行调整；
- 如果分区对应的响应为其他错误（比如Leader发生改变），那么加入到错误列表里；
- 最后，如果错误列表不为空，那么调用`handlePartitionsWithErrors()`处理错误。

### ConsumerFetcherThread

**handlePartitionsWithErrors**

```scala
def handlePartitionsWithErrors(partitions: Iterable[TopicAndPartition]) {
    //从当前ConsumerFetcherThread移除对应分区
    removePartitions(partitions.toSet) 
    //交给ConsumerFetcherManager去重新统筹调控
    consumerFetcherManager.addPartitionsWithError(partitions)
}
```

这里的其他错误一般就是Leader发生了改变，那么就需要`ConsumerFetcherManager`重新统筹调控由哪个`ConsumerFetcherThread`去拉取对应的分区数据了。

**handleOffsetOutOfRange**

如果发现无效的消费位移的话，根据配置委托`SimpleConsumer`重新去获取新的位移，看看源码：

```scala
def handleOffsetOutOfRange(topicAndPartition: TopicAndPartition): Long = {
    var startTimestamp : Long = 0
    config.autoOffsetReset match {
        case OffsetRequest.SmallestTimeString => startTimestamp = OffsetRequest.EarliestTime
        case OffsetRequest.LargestTimeString => startTimestamp = OffsetRequest.LatestTime
        case _ => startTimestamp = OffsetRequest.LatestTime
    }
    //委托SimpleConsumer重新去获取新的位移
    val newOffset = simpleConsumer.earliestOrLatestOffset(topicAndPartition, startTimestamp, Request.OrdinaryConsumerId)
    //设置ConsumerFetcherThread中的两个位移：拉取位移和消费位移
    val pti = partitionMap(topicAndPartition)
    pti.resetFetchOffset(newOffset)
    pti.resetConsumeOffset(newOffset)
    //返回给AbstractFetcherThread进行处理
    newOffset
}
```

**processPartitionData**

处理分区数据：将分区数据`ByteBufferMessageSet`加入到消费队列里面去，让KafkaStream去消费。

```scala
def processPartitionData(topicAndPartition: TopicAndPartition, fetchOffset: Long, partitionData: FetchResponsePartitionData) {
    val pti = partitionMap(topicAndPartition)
    if (pti.getFetchOffset != fetchOffset)
        throw new RuntimeException("Offset doesn't match for partition [%s,%d] pti offset: %d fetch offset: %d"
            .format(topicAndPartition.topic, topicAndPartition.partition, pti.getFetchOffset, fetchOffset))
    pti.enqueue(partitionData.messages.asInstanceOf[ByteBufferMessageSet])
}
```

### AbstractFetcherManager

`AbstractFetcherManager`与`AbstractFetcherThread`是配套使用的，`AbstractFetcherThread`负责具体的分区数据拉取逻辑，而`AbstractFetcherManager`作为Manger主要负责所有分区数据拉取工作的一个分配调度。它的使用有两个类：

- ConsumerFetcherManager：负责消费者拉取工作的分配调度
- ReplicaFetcherManager：负责副本复制拉取工作的调度分配

它内部维护了一个`AbstractFetcherThread`列表的映射关系，字段变量为：

```scala
val fetcherThreadMap = new HashMap[BrokerAndFetcherId, AbstractFetcherThread]
```

它还维护了一个重要的字段 **numFetchers**，这个字段控制为**每个Broker** 生成几个`AbstractFetcherThread`（即开启几个线程）来拉取分区数据。

每个`AbstractFetcherThread`对象都有一个唯一的 **FetcherId**，这个ID是如何生成的呢？看看源码：

```scala
private def getFetcherId(topic: String, partitionId: Int) : Int = {
    Utils.abs(31 * topic.hashCode() + partitionId) % numFetchers
}
```

所以，**FetcherId**的范围就在：`[0, numFetchers)`。

五个主要方法：

- createFetcherThread：抽象方法，需要子类实现
- addFetcherForPartitions：添加分区拉取任务
- removeFetcherForPartitions：移除分区拉取任务
- shutdownIdleFetcherThreads：停止所有空闲线程（没有分配到分区任务）
- closeAllFetchers：关闭所有拉取线程

**addFetcherForPartitions**

从**主题**、**分区**、**Broker**、**位移**四个纬度来添加分区拉取任务，如果这个分区任务应该被分配到的线程已经创建，那么直接加到该线程的任务列表里面去；否则就先创建线程，然后将任务加到线程的任务列表里面去。

直接上代码：

```scala
def addFetcherForPartitions(partitionAndOffsets: Map[TopicAndPartition, BrokerAndInitialOffset]) {
    mapLock synchronized {
        val partitionsPerFetcher = partitionAndOffsets.groupBy{ case(topicAndPartition, brokerAndInitialOffset) =>
            BrokerAndFetcherId(brokerAndInitialOffset.broker, getFetcherId(topicAndPartition.topic, topicAndPartition.partition))}
        for ((brokerAndFetcherId, partitionAndOffsets) <- partitionsPerFetcher) {
            var fetcherThread: AbstractFetcherThread = null
            fetcherThreadMap.get(brokerAndFetcherId) match {
                case Some(f) => fetcherThread = f
                case None =>
                    fetcherThread = createFetcherThread(brokerAndFetcherId.fetcherId, brokerAndFetcherId.broker)
                    fetcherThreadMap.put(brokerAndFetcherId, fetcherThread)
                    fetcherThread.start //开启拉取线程
            }
            fetcherThreadMap(brokerAndFetcherId).addPartitions(partitionAndOffsets.map { case (topicAndPartition, brokerAndInitOffset) =>
                topicAndPartition -> brokerAndInitOffset.initOffset
            })
        }
    }
}
```

**removeFetcherForPartitions**

从拉取线程的任务列表移除指定的分区任务，源码：

```scala
def removeFetcherForPartitions(partitions: Set[TopicAndPartition]) {
    mapLock synchronized {
        //遍历所有线程，无论任务在线程的任务列表里面与否，都执行下移除操作
        for ((key, fetcher) <- fetcherThreadMap) {
            fetcher.removePartitions(partitions)
        }
    }
    info("Removed fetcher for partitions %s".format(partitions.mkString(",")))
}
```

### ConsumerFetcherManager

我们先来看看它如何实现的父类抽象方法`createFetcherThread()`，代码很简单，就是直接新建一个`ConsumerFetcherThread`：

```scala
override def createFetcherThread(fetcherId: Int, sourceBroker: Broker): AbstractFetcherThread = {
    new ConsumerFetcherThread(
        "ConsumerFetcherThread-%s-%d-%d".format(consumerIdString, fetcherId, sourceBroker.id),
        config, sourceBroker, partitionMap, this)
}
```

还有一个逻辑需要实现，就是`ConsumerFetcherThread`在处理分区任务异常的时候会调用`ConsumerFetcherManager.addPartitionsWithError()`方法，我们看看调用代码：

```scala
def handlePartitionsWithErrors(partitions: Iterable[TopicAndPartition]) {
    //从当前ConsumerFetcherThread移除对应分区
    removePartitions(partitions.toSet) 
    //交给ConsumerFetcherManager去重新统筹调控
    consumerFetcherManager.addPartitionsWithError(partitions)
}
```

我们再来看看实现逻辑：

```scala
def addPartitionsWithError(partitionList: Iterable[TopicAndPartition]) {
    inLock(lock) {
        if (partitionMap != null) {
            noLeaderPartitionSet ++= partitionList
            cond.signalAll()
        }
    }
}
```

之前我们也说了，对于其他类型的异常来说，目前一般就只有分区Leader的改变导致的异常，所以在`ConsumerFetcherManager`里面需要重新获取这些分区的Leader，然后重新分配拉取任务！有个专门的线程`LeaderFinderThread`就是用于重新查找分区Leader这个工作的。

**LeaderFinderThread**

- 从ZK上拉取所有Brokers
- 委托`SyncProducer`发送`TopicMetadataRequest`请求，获取主题元数据
- 随机遍历所有Brokers直到找到对应的主题元数据
- 重置主题分区的Leader，然后调用`addFetcherForPartitions()`方法重启拉取任务





