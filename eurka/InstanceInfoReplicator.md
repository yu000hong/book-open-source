# InstanceInfoReplicator

如果实例某些属性发生了改变导致 **dirty** 状态，那么每隔一定时间会将实例重新注册到Eureka服务器；如果实例对应的这些属性没有变化，那么将不会向Eureka服务器发起注册请求。

哪些属性的改变会导致 **dirty** 状态：

- hostname
- ipAddress
- leaseInfo
- instanceStatus
- metadata

重新注册间隔时间：**instanceInfoReplicationIntervalSeconds**

核心方法：

```java
public void run() {
    try {
        //状态刷新，健康检查
        discoveryClient.refreshInstanceInfo();
        Long dirtyTimestamp = instanceInfo.isDirtyWithTime();
        //如果实例dirty了才会重新注册到Eureka服务器
        if (dirtyTimestamp != null) {
            discoveryClient.register();
            instanceInfo.unsetIsDirty(dirtyTimestamp);
        }
    } catch (Throwable t) {
        logger.warn("There was a problem with the instance info replicator", t);
    } finally {
        Future next = scheduler.schedule(this, replicationIntervalSeconds, TimeUnit.SECONDS);
        scheduledPeriodicRef.set(next);
    }
}
```

代码`discoveryClient.refreshInstanceInfo()`将会调用`HealthCheckHandler`进行实例的健康检查！
