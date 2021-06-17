# HealthCheckHandler

`InstanceInfoReplicator`会每隔一定时间(**instanceInfoReplicationIntervalSeconds**)调用`HealthCheckHandler`进行健康检查刷新实例状态，如果实例状态发生变化，会向Eureka服务器进行汇报。

接口定义：

```java
public interface HealthCheckHandler {

    InstanceInfo.InstanceStatus getStatus(InstanceInfo.InstanceStatus currentStatus);

}
```

### HealthCheckCallbackToHandlerBridge

目前只有一个实现类：`HealthCheckCallbackToHandlerBridge`，这是一个桥接类，将`HealthCheckCallback`类适配为`HealthCheckCallbackToHandlerBridge`。

```java
public class HealthCheckCallbackToHandlerBridge implements HealthCheckHandler {

    private final HealthCheckCallback callback;

    public HealthCheckCallbackToHandlerBridge() {
        callback = null;
    }

    public HealthCheckCallbackToHandlerBridge(HealthCheckCallback callback) {
        this.callback = callback;
    }

    @Override
    public InstanceInfo.InstanceStatus getStatus(InstanceInfo.InstanceStatus currentStatus) {
        if (null == callback || InstanceInfo.InstanceStatus.STARTING == currentStatus
                || InstanceInfo.InstanceStatus.OUT_OF_SERVICE == currentStatus) { // Do not go to healthcheck handler if the status is starting or OOS.
            return currentStatus;
        }

        return callback.isHealthy() ? InstanceInfo.InstanceStatus.UP : InstanceInfo.InstanceStatus.DOWN;
    }
}
```

### HealthCheckCallback

接口定义：

```java
public interface HealthCheckCallback {
    boolean isHealthy();
}
```
