# DiscoveryEvent

用在`DiscoveryClient`中的事件，有两个子类：

- CacheRefreshedEvent
- StatusChangeEvent

### CacheRefreshedEvent

从Eureka集群拉取了注册信息，更新了本地缓存之后触发的事件。

### StatusChangeEvent

Eureka实例有三个状态：

- status：当前状态，默认值为 **UP**，存储在`ApplicationInfoManager.InstanceInfo`里面；
- overriddenStatus：覆盖状态（TODO），默认值为 **UNKNOWN**，存储在`ApplicationInfoManager.InstanceInfo`里面；
- lastRemoteInstanceStatus：远程状态，默认值为 **UNKNOWN**，存储在`DiscoveryClient`里面；

如果Eureka实例 **shouldRegisterWithEureka** 设置为 **true**，那么它将会将自己也注册到Eureka集群，当其从Eureka集群拉取注册信息的时候也会将自己注册的状态拉取回来，这个状态就是：**远程状态**。

如果Eureka实例 **shouldRegisterWithEureka** 设置为 **false**，那么它不会将自己注册到Eureka集群，也就无法从Eureka服务器拉取到自己的远程状态，那么其远程状态将会一直保持 **UNKNOWN** 状态。

**什么时候会触发`StatusChangeEvent`？**

有两种情况会触发`StatusChangeEvent`，分别是：

- 当前状态发生变化：当Eureka实例本身状态发生改变的时候，就会触发事件，然后由`InstanceInfoRelicator`重新注册实例到Eureka集群；
- 远程状态发生变化：当从Eureka集群拉取注册信息后，发现前后两次拉取的远程状态发生改变了就会触发