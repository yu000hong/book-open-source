# ZKSessionExpireListener

如果在网络不佳或者ZK集群中某一台结点挂掉的情况下，有可能出现 **connection loss** 的情况，例如客户端和结点1连接断开，这时候客户端不需要任何的操作，只需要等待客户端与其他结点重新连接即可。这个过程可能导致两个结果：

1) 在会话超时之内连接成功：这个时候客户端成功切换到连接另一个结点，由于ZK在所有的结点上都同步了会话相关的数据，此时可以认为无缝迁移了；

2) 在会话超时之内没有连接上：这就是 **session expire** 的情况，这时候Zookeeper集群会认为会话已经结束，并清除和这个会话有关的所有数据，包括临时节点和注册的监视点Watcher。在会话超时之后，如果客户端重新连接上了Zookeeper集群，这个时候就会抛出 **session expired** 异常，此前会话的所有数据都没有了，需要我们手动重建。

从上面的描述，我们知道`ZKSessionExpireListener`就是为了重建由于会话失效导致被清掉的数据。我们看看代码：

```scala
class ZKSessionExpireListener(val dirs: ZKGroupDirs,
                              val consumerIdString: String,
                              val topicCount: TopicCount,
                              val loadBalancerListener: ZKRebalancerListener)
    extends IZkStateListener {

    @throws(classOf[Exception])
    def handleStateChanged(state: KeeperState) {
        // do nothing, since zkclient will do reconnect for us.
    }

    /**
     * Called after the zookeeper session has expired and a new session has been created. 
     * You would have to re-create any ephemeral nodes here.
     */
    @throws(classOf[Exception])
    def handleNewSession() {
        /**
         *  When we get a SessionExpired event, we lost all ephemeral nodes and zkclient has reestablished a
         *  connection for us. We need to release the ownership of the current consumer and re-register this
         *  consumer in the consumer registry and trigger a rebalance.
         */
        loadBalancerListener.resetState()
        registerConsumerInZK(dirs, consumerIdString, topicCount)
        loadBalancerListener.syncedRebalance()
    }

}
```

