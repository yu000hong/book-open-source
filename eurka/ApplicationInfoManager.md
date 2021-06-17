# ApplicationInfoManager

每一个Eureka实例都会由一个`ApplicationInfoManager`单例对象来管理其实例相关的一些信息，包含：

- InstanceInfo
- EurekaInstanceConfig
- InstanceStatusMapper
- StatusChangeListener
- refreshDataCenterInfoIfRequired()：重新获取 **hostname** 和 **ip** 两个属性
- refreshLeaseInfoIfRequired()：如果LeaseInfo和配置中的数据不一致，那么重新设置
- setInstanceStatus()：设置实例状态，触发`StatusChangeListener`
- registerAppMetadata()：设置元数据到 InstanceInfo 里面
