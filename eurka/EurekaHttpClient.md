# EurekaHttpClient

```java
EurekaHttpClient
   |-------------------------------HttpReplicationClient
   |                                       |
   |---AbstractJerseyEurekaHttpClient      |
   |                  |--------JerseyReplicationClient
   |                  |--------JerseyApplicationClient
   |---EurekaHttpClientDecorator
          |---MetricsCollectingEurekaHttpClient
          |---SessionedEurekaHttpClient
          |---RedirectingEurekaHttpClient
          |---RetryableEurekaHttpClient
```

`EurekaHttpClient`与Eureka服务器交互逻辑实现主要在`AbstractJerseyEurekaHttpClient`这个类里面，它虽然是个抽象类，但是它实现了与Eureka服务器进行HTTP请求的所有逻辑。`JerseyApplicationClient`直接继承并实现了抽象基类，`JerseyReplicationClient`继承了抽象基类，同时重写了`sendHeartBeat()`和`statusUpdate()`方法，并实现了`HttpReplicationClient`接口。

`EurekaHttpClientDecorator`及其4个子类都是为`EurekaHttpClient`增加一些其他附加功能：

- MetricsCollectingEurekaHttpClient：增加了指标收集功能
- SessionedEurekaHttpClient：增加了会话的概念，在会话超时后重新进行连接
- RedirectingEurekaHttpClient：增加了请求重定向功能
- RetryableEurekaHttpClient：增加了错误重试功能


### SessionedEurekaHttpClient

> `SessionedEurekaHttpClient` enforces full reconnect at a regular interval (a session), preventing a client to sticking to a particular Eureka server instance forever. This in turn guarantees even load distribution in case of cluster topology change.

`SessionedEurekaHttpClient`会每隔一段时间断开与Eureka服务器的连接，然后重新连接。这个会话时间由参数 **sessionedClientReconnectIntervalSeconds** 定义。

### RetryableEurekaHttpClient

> `RetryableEurekaHttpClient` retries failed requests on subsequent servers in the cluster. It maintains also simple quarantine list, so operations are not retried again on servers that are not reachable at the moment.
>
> All the servers to which communication failed are put on the quarantine list. First successful execution clears this list, which makes those server eligible for serving future requests. The list is also cleared once all available servers are exhausted.
>
> If 5xx status code is returned, `ServerStatusEvaluator` predicate evaluates if the retries should be retried on another server, or the response with this status code returned to the client.

`RetryableEurekaHttpClient`会在请求失败之后进行重试，从`ClusterResolver`中获取Eureka集群结点挨个重试。`RetryableEurekaHttpClient`它本身会记录一个失败结点列表，在重试的时候会避开失败结点。

> **那什么时候会清理失败结点呢，也就是对这些失败结点进行重试？**
>
> 如果失败结点超过配置`retryableClientQuarantineRefreshPercentage`指定的比例时，会清空失败结点列表，让这些失败结点可以有重试的机会。这个配置参数的默认值为 **0.66**，也即超过三分之二的结点都失败的时候进行清空重置操作。

### MetricsCollectingEurekaHttpClient

增加了指标收集功能

### RedirectingEurekaHttpClient

> `EurekaHttpClient` that follows redirect links, and executes the requests against the finally resolved endpoint. If registration and query requests must handled separately, two different instances shall be created.

允许对请求进行重定向，核心代码：

```java
private <R> EurekaHttpResponse<R> executeOnNewServer(RequestExecutor<R> requestExecutor,
                                                     AtomicReference<EurekaHttpClient> currentHttpClientRef) {
    URI targetUrl = null;
    for (int followRedirectCount = 0; followRedirectCount < MAX_FOLLOWED_REDIRECTS; followRedirectCount++) {
        EurekaHttpResponse<R> httpResponse = requestExecutor.execute(currentHttpClientRef.get());
        if (httpResponse.getStatusCode() != 302) {
            return httpResponse;
        }
        targetUrl = getRedirectBaseUri(httpResponse.getLocation());
        if (targetUrl == null) {
            throw new TransportException("Invalid redirect URL " + httpResponse.getLocation());
        }
        currentHttpClientRef.getAndSet(null).shutdown();
        currentHttpClientRef.set(factory.newClient(new DefaultEndpoint(targetUrl.toString())));
    }
    throw new TransportException("Follow redirect limit crossed for URI " + serviceEndpoint.getServiceUrl());
}
```

### AbstractJerseyEurekaHttpClient

这个类才是主要实现类，实现了对Eureka服务器的真实请求：

- register：`[POST] apps/$APP_NAME` 
- cancel：`[DELETE] apps/$APP_NAME/$ID`
- sendHeartBeat：`[PUT] apps/$APP_NAME/$ID`
- statusUpdate：`[PUT] apps/$APP_NAME/$ID/status`
- deleteStatusOverride：`[DELETE] apps/$APP_NAME/$ID/status`
- getApplication：`[GET] apps/$APP_NAME`
- getApplications：`[GET] apps/`
- getDelta：`[GET] apps/delta`
- getVip：`[GET] vips/$VIP_ADDRESS`
- getSecureVip：`[GET] svips/$SECURE_VIP_ADDRESS`
- getInstance：`[GET] instances/$ID`
- getInstance：`[GET] apps/$APP_NAME/$ID`

`AbstractJerseyEurekaHttpClient`有两个子实现类：

- JerseyApplicationClient：用于普通Eureka实例，直接继承
- JerseyReplicationClient：用于Eureka服务器之间进行复制（TODO），重写了`sendHeartBeat()`和`statusUpdate()`方法，并实现了`HttpReplicationClient`接口

