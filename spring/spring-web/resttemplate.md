# RestTemplate

继承关系：

```
HttpAccessor
    |
InterceptingHttpAccessor
    |
RestTemplate
```

涉及的类：

- ClientHttpRequest
- ClientHttpResponse
- ClientHttpRequestFactory
- ClientHttpRequestInitializer
- RequestCallback
- HttpMessageConverter
- ResponseErrorHandler
- ResponseExtractor
- UriTemplateHandler
- ClientHttpRequestExecution
- ClientHttpRequestInterceptor

我们先来看看`RestTemplate`的一些默认设置：

- 错误处理器：`DefaultResponseErrorHandler`
- 消息转换器列表：参见后面
- 请求拦截器列表：空
- 请求初始化器列表：空
- 请求对象工厂类：`SimpleClientHttpRequestFactory`
- 响应提取器：`ResponseEntityResponseExtractor`或`HttpMessageConverterExtractor`

注意：我们可以为每个请求单独停供**响应提取器**和**请求回调**这两个参数！


### ClientHttpRequest & ClientHttpResponse

`ClientHttpRequest`代表一个客户端请求，接口方法：

```java
public interface ClientHttpRequest extends HttpRequest, HttpOutputMessage {
    String getMethodValue();
    HttpMethod getMethod();
    HttpHeaders getHeaders();
    OutputStream getBody() throws IOException;

	ClientHttpResponse execute() throws IOException;
}
```

`ClientHttpResponse`代表一个客户端响应，接口方法：

```java
public interface ClientHttpResponse extends HttpInputMessage, Closeable {
    HttpStatus getStatusCode() throws IOException;
    int getRawStatusCode() throws IOException;
    String getStatusText() throws IOException;
    void close();
    HttpHeaders getHeaders();
    InputStream getBody() throws IOException;
}
```

### ClientHttpRequestFactory

`ClientHttpRequestFactory`是创建`ClientHttpRequest`的工厂类，接口方法：

```java
public interface ClientHttpRequestFactory {

	ClientHttpRequest createRequest(URI uri, HttpMethod httpMethod) throws IOException;

}
```

继承关系：

```java
ClientHttpRequestFactory
    |---MockMvcClientHttpRequestFactory
    |---SimpleClientHttpRequestFactory
    |---OkHttp3ClientHttpRequestFactory
    |---Netty4ClientHttpRequestFactory
    |---HttpComponentsClientHttpRequestFactory
    |       |---HttpComponentsAsyncClientHttpRequestFactory
    |---AbstractClientHttpRequestFactoryWrapper
            |---InterceptingClientHttpRequestFactory
            |---BufferingClientHttpRequestFactory
```

```java
ClientHttpRequest
   |---Netty4ClientHttpRequest
   |---MockClientHttpRequest
   |      |---MockAsyncClientHttpRequest
   |---AbstractClientHttpRequest
          |---SimpleStreamingClientHttpRequest
          |---HttpComponentsStreamingClientHttpRequest
          |---AbstractBufferingClientHttpRequest
                 |---BufferingClientHttpRequestWrapper
                 |---SimpleBufferingClientHttpRequest
                 |---InterceptingClientHttpRequest
                 |---HttpComponentsClientHttpRequest
                 |---OkHttp3ClientHttpRequest
```

```java
ClientHttpResponse
   |---AbstractClientHttpResponse
          |---SimpleClientHttpResponse
          |---Netty4ClientHttpResponse
          |---OkHttp3ClientHttpResponse
          |---HttpComponentsClientHttpResponse
          |---HttpComponentsAsyncClientHttpResponse
```

### AsyncClientHttpRequestFactory

`AsyncClientHttpRequestFactory`是创建`AsyncClientHttpRequest`的工厂类，接口方法：

```java
public interface AsyncClientHttpRequestFactory {

	AsyncClientHttpRequest createAsyncRequest(URI uri, HttpMethod httpMethod) throws IOException;

}
```

```java
AsyncClientHttpRequest
   |---MockAsyncClientHttpRequest
   |---AbstractAsyncClientHttpRequest
          |---Netty4ClientHttpRequest
          |---SimpleStreamingAsyncClientHttpRequest
          |---AbstractBufferingAsyncClientHttpRequest
                 |---SimpleBufferingAsyncClientHttpRequest
                 |---InterceptingAsyncClientHttpRequest
                 |---HttpComponentsAsyncClientHttpRequest
                 |---OkHttp3AsyncClientHttpRequest
```

### RequestCallback

> Callback interface for code that operates on a ClientHttpRequest. 
> Allows manipulating the request headers, and write to the request body.

RequestCallback接口方法：

```java
public interface RequestCallback {

	void doWithRequest(ClientHttpRequest request) throws IOException;

}
```

回调执行时机：在请求对象生成之后，真正执行调用之前。

### 请求执行逻辑


```java
/**
* doExecute()方法是真正的请求执行逻辑
*/
protected <T> T doExecute(URI url, @Nullable HttpMethod method, @Nullable RequestCallback requestCallback,
        @Nullable ResponseExtractor<T> responseExtractor) throws RestClientException {
    Assert.notNull(url, "URI is required");
    Assert.notNull(method, "HttpMethod is required");
    ClientHttpResponse response = null;
    try {
        //1. 创建请求对象
        ClientHttpRequest request = createRequest(url, method);
        //2. 执行请求回调
        if (requestCallback != null) {
            requestCallback.doWithRequest(request);
        }
        //3. 处理请求逻辑
        response = request.execute();
        //4. 响应错误处理
        handleResponse(url, method, response);
        //5. 提取响应数据
        return (responseExtractor != null ? responseExtractor.extractData(response) : null);
    }
    catch (IOException ex) {
        String resource = url.toString();
        String query = url.getRawQuery();
        resource = (query != null ? resource.substring(0, resource.indexOf('?')) : resource);
        throw new ResourceAccessException("I/O error on " + method.name() +
                " request for \"" + resource + "\": " + ex.getMessage(), ex);
    }
    finally {
        if (response != null) {
            response.close();
        }
    }
}
```

RestTemplate所有的请求调用方法最终都会调用`doExecute()`这个方法，这个方法包含了完整的请求执行逻辑，主要包含五个步骤：

1. 创建请求对象
2. 执行请求回调
3. 处理请求逻辑
4. 响应错误处理
5. 提取响应数据

##### 1. 创建请求对象

创建请求对象包含两个步骤：
- 使用工厂类创建请求对象
- 执行请求对象初始化过程

```java
protected ClientHttpRequest createRequest(URI url, HttpMethod method) throws IOException {
    ClientHttpRequest request = getRequestFactory().createRequest(url, method);
    this.clientHttpRequestInitializers.forEach(initializer -> initializer.initialize(request));
    return request;
}
```

##### 4. 响应错误处理

```java
protected void handleResponse(URI url, HttpMethod method, ClientHttpResponse response) throws IOException {
    ResponseErrorHandler errorHandler = getErrorHandler();
    boolean hasError = errorHandler.hasError(response);
    if (hasError) {
        errorHandler.handleError(url, method, response);
    }
}
```

##### 5. 提取响应数据

如果`responseExtractor`对象为null，那么直接返回null，否则使用它对响应进行处理，提取最终结果。

### ResponseExtractor

`ResponseExtractor`负责从响应对象中提取我们需要的结果，接口方法：

```java
public interface ResponseExtractor<T> {

	T extractData(ClientHttpResponse response) throws IOException;

}
```

主要的实现为：`HttpMessageConverterExtractor`，利用`HttpMessageConverter`进行解析转换。

代码如下：

```java
public T extractData(ClientHttpResponse response) throws IOException {
    MessageBodyClientHttpResponseWrapper responseWrapper = new MessageBodyClientHttpResponseWrapper(response);
    if (!responseWrapper.hasMessageBody() || responseWrapper.hasEmptyMessageBody()) {
        //如果没有响应体数据，那么返回null
        return null;
    }
    MediaType contentType = getContentType(responseWrapper);
    try {
        for (HttpMessageConverter<?> messageConverter : this.messageConverters) {
            if (messageConverter instanceof GenericHttpMessageConverter) {
                GenericHttpMessageConverter<?> genericMessageConverter =
                        (GenericHttpMessageConverter<?>) messageConverter;
                if (genericMessageConverter.canRead(this.responseType, null, contentType)) {
                    return (T) genericMessageConverter.read(this.responseType, null, responseWrapper);
                }
            }
            if (this.responseClass != null) {
                if (messageConverter.canRead(this.responseClass, contentType)) {
                    return (T) messageConverter.read((Class) this.responseClass, responseWrapper);
                }
            }
        }
    }
    catch (IOException | HttpMessageNotReadableException ex) {
        //如果读取响应体数据时发生异常，那么直接抛出RestClientException异常
        throw new RestClientException("Error while extracting response for type [" +
                this.responseType + "] and content type [" + contentType + "]", ex);
    }
    //如果无法解析响应体数据，那么抛出RestClientException异常
    throw new RestClientException("Could not extract response: no suitable HttpMessageConverter found " +
            "for response type [" + this.responseType + "] and content type [" + contentType + "]");
}
```

解析转换逻辑：

- 如果没有响应体数据，那么直接返回null
- 遍历所有的消息转换器，直到遇到能够转换响应体数据的消息转换器
    - 如果转换器支持范型类型的解析(实现了GenericHttpMessageConverter接口)，那么使用范型转换器相应的处理逻辑
    - 如果转换器时普通转换器，那么直接使用HttpMessageConverter接口对应的处理逻辑
- 如果读取响应体数据时发生异常，那么直接抛出RestClientException异常
- 如果无法解析响应体数据，那么抛出RestClientException异常

### ResponseErrorHandler

响应错误处理器接口，继承关系如下：

```java
    ResponseErrorHandler
            |
 DefaultResponseErrorHandler
            |
ExtractingResponseErrorHandler
```

接口方法：

```java
public interface ResponseErrorHandler {

	boolean hasError(ClientHttpResponse response) throws IOException;

	void handleError(ClientHttpResponse response) throws IOException;

	default void handleError(URI url, HttpMethod method, ClientHttpResponse response) throws IOException {
		handleError(response);
	}

}
```

##### DefaultResponseErrorHandler

`DefaultResponseErrorHandler`如何判断错误呢？答案是根据响应状态码，如果状态是**4xx**或**5xx**，
那么就表示响应出现了错误，需要执行错误处理逻辑。

`DefaultResponseErrorHandler`如何判断错误呢？答案是根据响应状态码。
如果状态是**4xx**，那么抛出`HttpClientErrorException`；
如果状态是**5xx**，那么抛出`HttpServerErrorException`。

##### ExtractingResponseErrorHandler

> Implementation of ResponseErrorHandler that uses HttpMessageConverters to convert HTTP error responses to RestClientExceptions.
>
> To use this error handler, you must specify a **status mapping** and/or a **series mapping**. 
> If either of these mappings has a match for the status code of a given ClientHttpResponse, `hasError(ClientHttpResponse)` will return true, and `handleError(ClientHttpResponse)` will attempt to use the configured message converters to convert the response into the mapped subclass of `RestClientException`. 
> Note that the status mapping takes precedence over series mapping.
>
> If there is no match, this error handler will default to the behavior of `DefaultResponseErrorHandler`. 
> Note that you can override this default behavior by specifying a series mapping from `HttpStatus.Series#CLIENT_ERROR` and/or `HttpStatus.Series#SERVER_ERROR` to null.

### UriTemplateHandler

继承关系如下：

```
UriTemplateHandler
   |---AbstractUriTemplateHandler
   |      |---DefaultUriTemplateHandler
   |---UriBuilderFactory
          |---DefaultUriBuilderFactory
```

接口方法如下：

```java
public interface UriTemplateHandler {
	URI expand(String uriTemplate, Map<String, ?> uriVariables);
	URI expand(String uriTemplate, Object... uriVariables);
}
```

关于URI的处理，最终都是委托给`UriComponentsBuilder`来完成。

我们在RestTemplate提供url的时候，可以使用`{name:[a-z]{1,5}}`（冒号后面时正则校验规则，可选值）的这种模板变量的方式，然后再提供变量键值对获得最终的URL。

我们先来看看其中的几个方法：

```java
public <T> T getForObject(String url, Class<T> responseType, Object... uriVariables) throws RestClientException;
public <T> T getForObject(String url, Class<T> responseType, Map<String, ?> uriVariables) throws RestClientException;
public <T> ResponseEntity<T> getForEntity(String url, Class<T> responseType, Object... uriVariables) throws RestClientException;
public <T> ResponseEntity<T> getForEntity(String url, Class<T> responseType, Map<String, ?> uriVariables) throws RestClientException;
```

我们提供变量值的时候，可以有两种方式。一种方式是作为位置变量，提供一个值数组；另一种方式是提供键值对字典。

示例：

URL：`http://{server}/{service}?server={server}&name={name:[a-z]{1,5}}`

值数组方式：

```json
["baidu.com", "search", "baidu.com", "hadoop"]
```

键值对方式：

```json
{
    "name": "hadoop",
    "server": "baidu.com",
    "service": "search"
}
```

最终URL：**http://baidu.com/search?server=baidu.com&name=hadoop**

### 消息转换器

消息转换器由接口`HttpMessageConverter`承载。

RestTemplate默认会加载如下消息转换器：

- ByteArrayHttpMessageConverter
- StringHttpMessageConverter
- ResourceHttpMessageConverter
- SourceHttpMessageConverter
- AllEncompassingFormHttpMessageConverter
- AtomFeedHttpMessageConverter(按需加载)
- RssChannelHttpMessageConverter(按需加载)
- MappingJackson2XmlHttpMessageConverter(按需加载)
- Jaxb2RootElementHttpMessageConverter(按需加载)
- MappingJackson2HttpMessageConverter(按需加载)
- GsonHttpMessageConverter(按需加载)
- JsonbHttpMessageConverter(按需加载)
- MappingJackson2SmileHttpMessageConverter(按需加载)
- MappingJackson2CborHttpMessageConverter(按需加载)

`(按需加载)`：是指对应的类库是否在类搜索路径下，如果在的话就加载，否则不加载。

TODO：需要详细说明每个消息转换器的作用，支持的**MediaTypt**媒体类型。

### 请求拦截器

涉及的类有：

- ClientHttpRequestInterceptor
- InterceptingClientHttpRequestFactory
- InterceptingClientHttpRequest
- ClientHttpRequestExecution

`RestTemplate`继承自`InterceptingHttpAccessor`，`InterceptingHttpAccessor`父类已经处理了请求拦截相关的逻辑。
如果我们设置了拦截器，那么在设置请求对象工厂类的时候最终会生成一个带拦截功能的`InterceptingClientHttpRequestFactory`，
请求对象创建功能代理给了我们要设置的请求对象工厂类。那么拦截器在什么时候执行呢？我们先看看代码：

```java
public ClientHttpResponse execute(HttpRequest request, byte[] body) throws IOException {
    if (this.iterator.hasNext()) {
        //先依次执行完所有的拦截器拦截逻辑
        ClientHttpRequestInterceptor nextInterceptor = this.iterator.next();
        return nextInterceptor.intercept(request, body, this);
    }
    else {
        //最后再执行请求逻辑
        //将请求头和请求体传递给代理，由代理执行最终的请求逻辑
        HttpMethod method = request.getMethod();
        ClientHttpRequest delegate = requestFactory.createRequest(request.getURI(), method);
        request.getHeaders().forEach((key, value) -> delegate.getHeaders().addAll(key, value));
        if (body.length > 0) {
            if (delegate instanceof StreamingHttpOutputMessage) {
                StreamingHttpOutputMessage streamingOutputMessage = (StreamingHttpOutputMessage) delegate;
                streamingOutputMessage.setBody(outputStream -> StreamUtils.copy(body, outputStream));
            }
            else {
                StreamUtils.copy(body, delegate.getBody());
            }
        }
        return delegate.execute();
    }
}
```

可以看到，拦截器的拦截逻辑的执行是先于请求执行逻辑的。



### 参考资源

[RestTemplate组件：ClientHttpRequestFactory、ClientHttpRequestInterceptor、ResponseExtractor【享学Spring MVC】](https://blog.csdn.net/f641385712/article/details/100713622)

[Guide to UriComponentsBuilder in Spring](https://www.baeldung.com/spring-uricomponentsbuilder)



