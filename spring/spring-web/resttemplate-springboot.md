# RestTemplate相关增强

**spring-boot**

- RootUriTemplateHandler
- RestTemplateCustomizer
- ClientHttpRequestFactorySupplier
- RestTemplateBuilder

**spring-boot-autoconfigure**

- RestTemplateAutoConfiguration

**spring-boot-actuator**

- RestTemplateExchangeTags
- RestTemplateExchangeTagsProvider
- DefaultRestTemplateExchangeTagsProvider
- MetricsRestTemplateCustomizer
- MetricsClientHttpRequestInterceptor

**spring-boot-actuator-autoconfigure**

- HttpClientMetricsAutoConfiguration
- RestTemplateMetricsConfiguration
- WebClientMetricsConfiguration

### RootUriTemplateHandler

我们知道，`UriTemplateHandler`也即URI的处理最终都是委托给`UriComponentsBuilder`来完成的。`RootUriTemplateHandler`可以设置一个根URI，然后基于这个根URL计算绝对的完整的URL。

```java
private String apply(String uriTemplate) {
    //如果uri以"/"开头，那么将会附上根URI这个前缀
    if (StringUtils.startsWithIgnoreCase(uriTemplate, "/")) {
        return getRootUri() + uriTemplate;
    }
    //否则，直接返回
    return uriTemplate;
}
```

### ClientHttpRequestFactorySupplier

它作为Supplier，能够返回一个`ClientHttpRequestFactoryS`对象：

- 如果`HttpComponentsClientHttpRequestFactory`存在于类搜索路径上，那么返回对应的实例；
- 如果`OkHttp3ClientHttpRequestFactory`存在于类搜索路径上，那么返回对应的实例；
- 如果都不存在，那么返回`SimpleClientHttpRequestFactory`实例。

### RestTemplateCustomizer

Spring实现自定义化控制的Cutomizer接口，主要是提供给`RestTemplateBuilder`构建类使用：

```java
public interface RestTemplateCustomizer {

	void customize(RestTemplate restTemplate);

}
```

### RestTemplateBuilder

TODO

### RestTemplateAutoConfiguration

这个类很简单，代码只有十多行，主要作用就是根据已经配置的`HttpMessageConverters`和`RestTemplateCustomizer`来生成一个`RestTemplateBuilder`实例Bean：

```java
@Configuration
@AutoConfigureAfter(HttpMessageConvertersAutoConfiguration.class)
@ConditionalOnClass(RestTemplate.class)
public class RestTemplateAutoConfiguration {

	private final ObjectProvider<HttpMessageConverters> messageConverters;

	private final ObjectProvider<RestTemplateCustomizer> restTemplateCustomizers;

	public RestTemplateAutoConfiguration(
			ObjectProvider<HttpMessageConverters> messageConverters,
			ObjectProvider<RestTemplateCustomizer> restTemplateCustomizers) {
		this.messageConverters = messageConverters;
		this.restTemplateCustomizers = restTemplateCustomizers;
	}

	@Bean
	@ConditionalOnMissingBean
	public RestTemplateBuilder restTemplateBuilder() {
		RestTemplateBuilder builder = new RestTemplateBuilder();
		HttpMessageConverters converters = this.messageConverters.getIfUnique();
		if (converters != null) {
			builder = builder.messageConverters(converters.getConverters());
		}

		List<RestTemplateCustomizer> customizers = this.restTemplateCustomizers
				.orderedStream().collect(Collectors.toList());
		if (!CollectionUtils.isEmpty(customizers)) {
			builder = builder.customizers(customizers);
		}
		return builder;
	}

}
```

### RestTemplateExchangeTags

这个一个工具类，只有一些静态工具方法，如：

```java
//生成method标签
public static Tag method(HttpRequest request) {
    return Tag.of("method", request.getMethod().name());
}
//生成uri标签
public static Tag uri(HttpRequest request) {
    return Tag.of("uri", ensureLeadingSlash(stripUri(request.getURI().toString())));
}
//生成status标签
public static Tag status(ClientHttpResponse response) {
    return Tag.of("status", getStatusMessage(response));
}
//生成clientName标签
public static Tag clientName(HttpRequest request) {
    String host = request.getURI().getHost();
    if (host == null) {
        host = "none";
    }
    return Tag.of("clientName", host);
}
```

### DefaultRestTemplateExchangeTagsProvider

默认实现，提供了**method**、**uri**、**status**和**clientName**四个标签，代码如下：

```java
public class DefaultRestTemplateExchangeTagsProvider
		implements RestTemplateExchangeTagsProvider {

	@Override
	public Iterable<Tag> getTags(String urlTemplate, HttpRequest request,
			ClientHttpResponse response) {
		Tag uriTag = (StringUtils.hasText(urlTemplate)
				? RestTemplateExchangeTags.uri(urlTemplate)
				: RestTemplateExchangeTags.uri(request));
		return Arrays.asList(RestTemplateExchangeTags.method(request), uriTag,
				RestTemplateExchangeTags.status(response),
				RestTemplateExchangeTags.clientName(request));
	}

}
```

### MetricsClientHttpRequestInterceptor

拦截每个请求，对每个请求进行耗时统计，使用的是Micrometer的Timer指标。并且使用`DefaultRestTemplateExchangeTagsProvider`为每个请求打上特定的四个标签。

我们看看拦截处理代码：

```java
@Override
public ClientHttpResponse intercept(HttpRequest request, byte[] body,
        ClientHttpRequestExecution execution) throws IOException {
    long startTime = System.nanoTime();
    ClientHttpResponse response = null;
    try {
        response = execution.execute(request, body);
        return response;
    }
    finally {
        getTimeBuilder(request, response).register(this.meterRegistry)
                .record(System.nanoTime() - startTime, TimeUnit.NANOSECONDS);
        urlTemplate.remove();
    }
}
```

### MetricsRestTemplateCustomizer

为RestTemplate增加`MetricsClientHttpRequestInterceptor`拦截器，以提供耗时指标统计功能。

### HttpClientMetricsAutoConfiguration

自动配置类`HttpClientMetricsAutoConfiguration`引入了两个配置类：

- RestTemplateMetricsConfiguration：同步RestTemplate的指标配置
- WebClientMetricsConfiguration：异步WebClient的指标配置

我们这里暂时先忽略异步WebClient的配置，来看看同步RestTemplate的指标配置。

`RestTemplateMetricsConfiguration`自动生成了两个类：

- DefaultRestTemplateExchangeTagsProvider：为每个请求打上4个标签
- MetricsRestTemplateCustomizer：为RestTemplate注入指标搜集拦截器

> **注意**：这里需要注意的一点是异步WebClient和同步RestTemplate都是使用的同一个指标名称**http.client.requests**，这个指标名称可以由参数`management.metrics.web.client.requests-metric-name`进行配置。

代码很简单：

```java
@Configuration
@ConditionalOnClass(RestTemplate.class)
class RestTemplateMetricsConfiguration {

	private final MetricsProperties properties;

	RestTemplateMetricsConfiguration(MetricsProperties properties) {
		this.properties = properties;
	}

	@Bean
	@ConditionalOnMissingBean(RestTemplateExchangeTagsProvider.class)
	public DefaultRestTemplateExchangeTagsProvider restTemplateTagConfigurer() {
		return new DefaultRestTemplateExchangeTagsProvider();
	}

	@Bean
	public MetricsRestTemplateCustomizer metricsRestTemplateCustomizer(
			MeterRegistry meterRegistry,
			RestTemplateExchangeTagsProvider restTemplateTagConfigurer) {
		return new MetricsRestTemplateCustomizer(meterRegistry, restTemplateTagConfigurer,
				this.properties.getWeb().getClient().getRequestsMetricName());
	}

}
```