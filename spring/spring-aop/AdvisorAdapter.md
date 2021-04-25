# AdvisorAdapter

    AOP联盟定义了三种类型的拦截器：

    - MethodInterceptor
    - ConstructorInterceptor
    - FieldInterceptor

    Spring AOP框架只支持方法调用拦截器：MethodInterceptor。


`AdvisorAdapter`使用了适配器模式使得Spring AOP框架能够处理AOP联盟提供的通知类型。AOP联盟提供的通知Advice接口是一个标记接口，而Spring AOP框架中最终都是利用`MethodInterceptor`来处理通知，执行通知代码的。所以，需要AdvisorAdapter这样一个适配器类，将`Advice`转化成`MethodInterceptor`。


```java
public interface AdvisorAdapter {
	boolean supportsAdvice(Advice advice);
	MethodInterceptor getInterceptor(Advisor advisor);
}
```

目前，Spring AOP框架提供了三种类型的适配器类：

- MethodBeforeAdviceAdapter：`MethodBeforeAdvice` -> `MethodBeforeAdviceInterceptor`
- AfterReturningAdviceAdapter：`AfterReturningAdvice` -> `AfterReturningAdviceInterceptor`
- ThrowsAdviceAdapter：`ThrowsAdvice` -> `ThrowsAdviceInterceptor`

**AdvisorAdapterRegistry**

`AdvisorAdapterRegistry`提供了注册器，同时可以将`Advice`转化为`Advisor`，将`Advisor`转化为`MethodInterceptor`。

```java
public interface AdvisorAdapterRegistry {

	Advisor wrap(Object advice) throws UnknownAdviceTypeException;

	MethodInterceptor[] getInterceptors(Advisor advisor) throws UnknownAdviceTypeException;

	void registerAdvisorAdapter(AdvisorAdapter adapter);

}
```

**DefaultAdvisorAdapterRegistry**

我们看看仅有的实现：

```java
public class DefaultAdvisorAdapterRegistry implements AdvisorAdapterRegistry {
	private final List<AdvisorAdapter> adapters = new ArrayList<>(3);

	public DefaultAdvisorAdapterRegistry() {
		registerAdvisorAdapter(new MethodBeforeAdviceAdapter());
		registerAdvisorAdapter(new AfterReturningAdviceAdapter());
		registerAdvisorAdapter(new ThrowsAdviceAdapter());
	}

	@Override
	public Advisor wrap(Object adviceObject) throws UnknownAdviceTypeException {
		if (adviceObject instanceof Advisor) {
			return (Advisor) adviceObject;
		}
		if (!(adviceObject instanceof Advice)) {
			throw new UnknownAdviceTypeException(adviceObject);
		}
		Advice advice = (Advice) adviceObject;
		if (advice instanceof MethodInterceptor) {
			// So well-known it doesn't even need an adapter.
			return new DefaultPointcutAdvisor(advice);
		}
		for (AdvisorAdapter adapter : this.adapters) {
			// Check that it is supported.
			if (adapter.supportsAdvice(advice)) {
				return new DefaultPointcutAdvisor(advice);
			}
		}
		throw new UnknownAdviceTypeException(advice);
	}

	@Override
	public MethodInterceptor[] getInterceptors(Advisor advisor) throws UnknownAdviceTypeException {
		List<MethodInterceptor> interceptors = new ArrayList<>(3);
		Advice advice = advisor.getAdvice();
		if (advice instanceof MethodInterceptor) {
			interceptors.add((MethodInterceptor) advice);
		}
		for (AdvisorAdapter adapter : this.adapters) {
			if (adapter.supportsAdvice(advice)) {
				interceptors.add(adapter.getInterceptor(advisor));
			}
		}
		if (interceptors.isEmpty()) {
			throw new UnknownAdviceTypeException(advisor.getAdvice());
		}
		return interceptors.toArray(new MethodInterceptor[0]);
	}

	@Override
	public void registerAdvisorAdapter(AdvisorAdapter adapter) {
		this.adapters.add(adapter);
	}

}
```

如果想将自定义或第三方`Advice`应用于Spring AOP框架中，那么就必须使用对应的`AdvisorAdapter`，同时还得注册到`AdvisorAdapterRegistry`中去。注册的工作完全可以交予`AdvisorAdapterRegistrationManager`，下面我们看看这个类。

**AdvisorAdapterRegistrationManager**

`AdvisorAdapterRegistrationManager`实现了`BeanPostProcessor`，在检测到`AdvisorAdapter`对象时会将其注册到`AdvisorAdapterRegistry`中去。所以，如果需要适配第三方或自定义的`Advice`的时候，我们只需要定义对应的适配器Bean对象即可。

```java
public class AdvisorAdapterRegistrationManager implements BeanPostProcessor {

	private AdvisorAdapterRegistry advisorAdapterRegistry = GlobalAdvisorAdapterRegistry.getInstance();

	public void setAdvisorAdapterRegistry(AdvisorAdapterRegistry advisorAdapterRegistry) {
		this.advisorAdapterRegistry = advisorAdapterRegistry;
	}

	@Override
	public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		if (bean instanceof AdvisorAdapter){
			this.advisorAdapterRegistry.registerAdvisorAdapter((AdvisorAdapter) bean);
		}
		return bean;
	}

}
```
