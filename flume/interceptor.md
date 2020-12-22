# Interceptor

官方文档：[Flume Interceptors](https://flume.apache.org/releases/content/1.9.0/FlumeUserGuide.html#flume-interceptors)

```java
interface Interceptor {

  /**
   * Return Original or modified event, or null if the Event is to be dropped.
   */
  Event intercept(Event event);

  /**
   * The size of output list MUST NOT BE GREATER than the size of the input list 
   * (transformation and removal ONLY).
   */
  List<Event> intercept(List<Event> events);

  void initialize();
  void close();
}
```

`Interceptor` 是一个接口，用于拦截消息事件`Event`，对消息事件进行过滤或做一些变换操作。

- `initialize()`: 生命周期方法，初始化拦截器
- `close()`: 生命周期方法，关闭拦截器
- `intercept()`: 拦截消息事件，注意拦截器智能进行转换或过滤操作，不能增加消息事件，因为(TODO)

`Interceptor` 实现类有：

- StaticInterceptor
- RemoveHeaderInterceptor
- TimestampInterceptor
- HostInterceptor
- UUIDInterceptor
- SearchAndReplaceInterceptor
- RegexFilteringInterceptor
- RegexExtractorInterceptor
- MorphlineInterceptor
- InterceptorChain

`StaticInterceptor`, `RemoveHeaderInterceptor`, `TimestampInterceptor`, `HostInterceptor`, `UUIDInterceptor` 是处理Header的拦截器；`SearchAndReplaceInterceptor`, `RegexFilteringInterceptor`, `RegexExtractorInterceptor`, `MorphlineInterceptor` 是处理Body的拦截器；`InterceptorChain` 使用责任链模式，将Source配置的所有拦截器按序执行拦截，返回最后的结果。


