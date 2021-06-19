# ZKTopicPartitionChangeListener

代码很简单，主要功能就是监听 **/brokers/topics/$TOPIC**，如果主题元数据发生了变化，那么会触发消费重平衡计算。我们消费者消费了多少个主题，对应的就会在每个主题路径下注册该监听器。

代码如下：

```scala
class ZKTopicPartitionChangeListener(val loadBalancerListener: ZKRebalancerListener) extends IZkDataListener {

    def handleDataChange(dataPath : String, data: Object) {
        try {
            // queue up the rebalance event
            loadBalancerListener.rebalanceEventTriggered()
            // There is no need to re-subscribe the watcher since it will be automatically
            // re-registered upon firing of this event by zkClient
        } catch {
            case e: Throwable => error("Error while handling topic partition change for data path " + dataPath, e )
        }
    }

    @throws(classOf[Exception])
    def handleDataDeleted(dataPath : String) {
        // TODO: This need to be implemented when we support delete topic
        warn("Topic for path " + dataPath + " gets deleted, which should not happen at this time")
    }
}
```
