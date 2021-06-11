# Eureka 实例

**Instance**：表示注册到 Eureka Server 的一个实例。

### EurekaInstanceConfig

`EurekaInstanceConfig`是一个接口，包含注册实例的一些配置信息：

- instanceId：实例ID
- appname：应用名称，由`spring.application.name`指定
- hostname：
- ipAddress：
- nonSecurePort：普通端口，默认值80
- securePort：安全端口，默认值443
- nonSecurePortEnabled：普通端口是否启用
- securePortEnabled：安全端口是否启用
- virtualHostName：
- secureVirtualHostName：
- appGroupName：应用组名，好像没啥用
- leaseRenewalIntervalInSeconds：实例多久发送一次心跳
- leaseExpirationDurationInSeconds：Eureka服务器多长时间没接受到心跳时，就会将实例剔除
- instanceEnabledOnInit：是否在实例注册到Eureka服务器之后即可接受请求
- metadataMap：一些元数据
- homePageUrlPath：**/**
- statusPageUrlPath：**/actuator/info**
- healthCheckUrlPath：**/actuator/healty**
- dataCenterInfo：AWS相关属性
- asgName：AWS相关属性

### InstanceStatus

注册实例的状态，枚举类型，有如下值：

- UP：实例状态正常
- DOWN：健康检查失败
- STARTING：正在启动，不能接受请求
- OUT_OF_SERVICE：暂时无法提供服务，无法接受请求
- UNKNOWN：未知状态

### InstanceInfo

实例相关的信息，Eureka服务器会将所有已经注册的实例以`InstanceInfo`对象缓存在内存中。


### ApplicationInfoManager

每一个Eureka实例都会由一个`ApplicationInfoManager`单例对象来管理其实例相关的一些信息，包含：

- InstanceInfo
- EurekaInstanceConfig
- InstanceStatusMapper
- StatusChangeListener
- refreshDataCenterInfoIfRequired()：重新获取 **hostname** 和 **ip** 两个属性
- refreshLeaseInfoIfRequired()：如果LeaseInfo和配置中的数据不一致，那么重新设置
- setInstanceStatus()：设置实例状态，触发`StatusChangeListener`
- registerAppMetadata()：设置元数据到 InstanceInfo 里面

### InstanceInfoReplicator

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

### HealthCheckHandler

如上面的代码，`InstanceInfoReplicator`会每隔一定时间(**instanceInfoReplicationIntervalSeconds**)调用`HealthCheckHandler`进行健康检查刷新实例状态，如果实例状态发生变化，会向Eureka服务器进行汇报。

### EurekaClientConfig

- registryFetchIntervalSeconds
- instanceInfoReplicationIntervalSeconds
- initialInstanceInfoReplicationIntervalSeconds
- eurekaServiceUrlPollIntervalSeconds
- shouldGZipContent
- eurekaServerReadTimeoutSeconds
- eurekaServerConnectTimeoutSeconds
- eurekaServerTotalConnections
- eurekaServerTotalConnectionsPerHost
- eurekaServerURLContext
- eurekaServerPort
- eurekaServerDNSName
- shouldUseDnsForFetchingServiceUrls
- shouldRegisterWithEureka
- shouldUnregisterOnShutdown
- shouldPreferSameZoneEureka
- backupRegistryImpl
- allowRedirects
- shouldFetchRegistry
- shouldEnforceFetchRegistryAtInit
- shouldEnforceRegistrationAtInit
- shouldLogDeltaDiff
- shouldDisableDelta
- shouldFilterOnlyUpInstances
- shouldOnDemandUpdateStatusChange
- eurekaConnectionIdleTimeoutSeconds
- region
- heartbeatExecutorThreadPoolSize
- heartbeatExecutorExponentialBackOffBound
- cacheRefreshExecutorThreadPoolSize
- cacheRefreshExecutorExponentialBackOffBound
- proxyHost
- proxyPort
- roxyUserName
- proxyPassword

