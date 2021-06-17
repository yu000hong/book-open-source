# ClusterResolver

集群解析器，返回Eureka集群某个Region内所有Zone的所有服务器结点。

```java
public interface ClusterResolver<T extends EurekaEndpoint> {

    String getRegion();

    List<T> getClusterEndpoints();

}
```

继承关系：

```java
ClusterResolver
   |---EurekaHttpResolver
   |---ConfigClusterResolver
   |---ApplicationsResolver
   |---StaticClusterResolver
   |---ZoneAffinityClusterResolver
   |---ReloadingClusterResolver
   |---DnsClusterResolver
   |---DnsTxtRecordClusterResolver
   |---ClosableResolver
          |---AsyncResolver
```

### EurekaHttpResolver

通过HTTP调用从其他服务器中获取，核心代码：

```java
client = clientFactory.newClient();
EurekaHttpResponse<Applications> response = client.getVip(vipAddress);
if (validResponse(response)) {
    Applications applications = response.getEntity();
    if (applications != null) {
        applications.shuffleInstances(true);  // filter out non-UP instances
        List<InstanceInfo> validInstanceInfos = applications.getInstancesByVirtualHostName(vipAddress);
        for (InstanceInfo instanceInfo : validInstanceInfos) {
            AwsEndpoint endpoint = ResolverUtils.instanceInfoToEndpoint(clientConfig, transportConfig, instanceInfo);
            if (endpoint != null) {
                result.add(endpoint);
            }
        }
        logger.debug("Retrieved endpoint list {}", result);
        return result;
    }
}
```

### ApplicationsResolver

从`ApplicationsSource`中获取Eureka集群机器信息，核心代码：

```java
public List<AwsEndpoint> getClusterEndpoints() {
    List<AwsEndpoint> result = new ArrayList<>();

    Applications applications = applicationsSource.getApplications(
            transportConfig.getApplicationsResolverDataStalenessThresholdSeconds(), TimeUnit.SECONDS);

    if (applications != null && vipAddress != null) {
        List<InstanceInfo> validInstanceInfos = applications.getInstancesByVirtualHostName(vipAddress);
        for (InstanceInfo instanceInfo : validInstanceInfos) {
            if (instanceInfo.getStatus() == InstanceInfo.InstanceStatus.UP) {
                AwsEndpoint endpoint = ResolverUtils.instanceInfoToEndpoint(clientConfig, transportConfig, instanceInfo);
                if (endpoint != null) {
                    result.add(endpoint);
                }
            }
        }
    }
    return result;
}
```

与`EurekaHttpResolver`有点类似，都是先获取所有注册应用信息，然后根据 **vipAddress** 去查询对应的Eureka集群结点。

### StaticClusterResolver

静态配置Eureka集群所有机器，直接硬编码的方式设置：

```java
public class StaticClusterResolver<T extends EurekaEndpoint> implements ClusterResolver<T> {
    private final List<T> eurekaEndpoints;
    private final String region;

    public StaticClusterResolver(String region, List<T> eurekaEndpoints) {
        this.eurekaEndpoints = eurekaEndpoints;
        this.region = region;
    }

    @Override
    public String getRegion() {
        return region;
    }

    @Override
    public List<T> getClusterEndpoints() {
        return eurekaEndpoints;
    }
}
```

### ConfigClusterResolver

从配置中获取Eureka集群所有机器，有两种配置：

- 从DNS中获取
- 从配置中获取

```java
public List<AwsEndpoint> getClusterEndpoints() {
    if (clientConfig.shouldUseDnsForFetchingServiceUrls()) {
        //从DNS中获取
        return getClusterEndpointsFromDns();
    } else {
        //从配置中获取
        return getClusterEndpointsFromConfig();
    }
}
```

```java
private List<AwsEndpoint> getClusterEndpointsFromDns() {
    String discoveryDnsName = getDNSName();
    int port = Integer.parseInt(clientConfig.getEurekaServerPort());
    //直接使用DnsTxtRecordClusterResolver
    DnsTxtRecordClusterResolver dnsResolver = new DnsTxtRecordClusterResolver(
            getRegion(),
            discoveryDnsName,
            true,
            port,
            false,
            clientConfig.getEurekaServerURLContext()
    );
    return dnsResolver.getClusterEndpoints();
}
```

```java
private List<AwsEndpoint> getClusterEndpointsFromConfig() {
    String[] availZones = clientConfig.getAvailabilityZones(clientConfig.getRegion());
    String myZone = InstanceInfo.getZone(availZones, myInstanceInfo);
    Map<String, List<String>> serviceUrls = EndpointUtils
            .getServiceUrlsMapFromConfig(clientConfig, myZone, clientConfig.shouldPreferSameZoneEureka());
    List<AwsEndpoint> endpoints = new ArrayList<>();
    for (String zone : serviceUrls.keySet()) {
        for (String url : serviceUrls.get(zone)) {
            try {
                //根据serviceUrl来发现集群其他机器的！
                endpoints.add(new AwsEndpoint(url, getRegion(), zone));
            } catch (Exception ignore) {
            }
        }
    }
    return endpoints;
}
```

### ZoneAffinityClusterResolver

> It is a cluster resolver that reorders the server list, such that the first server on the list is in the same zone as the client. The server is chosen randomly from the available pool of server in that zone. The remaining servers are appended in a random order, local zone first, followed by servers from other zones.

其他`ClusterResovler`返回的集群结点包含了当前Region内所有Zone的结点，没有特定的顺序，而`ZoneAffinityClusterResolver`可以根据 **zoneAffinity** 的值来决定是否把在同一Zone的结点放在列表的最开始或者放在最后面：

- zoneAffinity=true：将同一Zone的集群结点放在列表最开始
- zoneAffinity=false：将同一Zone的集群结点放在列表最后面

### ReloadingClusterResolver

> A cluster resolver implementation that periodically creates a new `ClusterResolver` instance that swaps the previous value. If the new resolver cannot be created or contains empty server list, the previous one is used. Failed requests are retried using exponential back-off strategy.

TODO

### DnsClusterResolver

TODO

### DnsTxtRecordClusterResolver

TODO

### AsyncResolver

> An async resolver that keeps a cached version of the endpoint list value for gets, and updates this cache periodically in a different thread.

异步集群解析器，它会缓存之前的解析结果，并且定期 **eurekaServiceUrlPollIntervalSeconds** 调用委托解析器去更新缓存。

