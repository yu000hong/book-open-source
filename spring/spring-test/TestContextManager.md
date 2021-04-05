# TestContextManager

### TestExecutionListener

`TestExecutionListener`使用观察者模式监听由`TestContextManager`发布的相关事件，包括：

- prepareTestInstance
- beforeTestClass
- afterTestClass
- beforeTestMethod
- afterTestMethod
- beforeTestExecution
- afterTestExecution

`TestExecutionListener`接口代码如下：

```java
public interface TestExecutionListener {
    void prepareTestInstance(TestContext testContext);
    void beforeTestClass(TestContext testContext);
    void afterTestClass(TestContext testContext);
    void beforeTestMethod(TestContext testContext);
    void afterTestMethod(TestContext testContext);
    void beforeTestExecution(TestContext testContext);
    void afterTestExecution(TestContext testContext);
}
```

### TestContext

`TestContext`是一个接口，每个测试方法对应一个这样的实例，接口代码：

```java
public interface TestContext extends AttributeAccessor, Serializable {
    boolean hasApplicationContext();
    ApplicationContext getApplicationContext();
    void publishEvent(Function<TestContext, ? extends ApplicationEvent> eventFactory);
    Class<?> getTestClass();
    Object getTestInstance();
    Method getTestMethod();
    Throwable getTestException();
    void markApplicationContextDirty(HierarchyMode hierarchyMode);
    void updateState(Object testInstance, Method testMethod, Throwable testException);
}
```

### TestContextManager

> `TestContextManager` is the main entry point into the **Spring TestContext Framework**.
> 
> `TestContextManager` is responsible for managing a single `TestContext` and signaling events to each registered `TestExecutionListener` at the following test execution points.

我们来看看`TestContextManager`如何构建`TestContext`的：

```java
public TestContextManager(TestContextBootstrapper testContextBootstrapper) {
    this.testContext = testContextBootstrapper.buildTestContext();
    registerTestExecutionListeners(testContextBootstrapper.getTestExecutionListeners());
}
public TestContextManager(Class<?> testClass) {
    this(BootstrapUtils.resolveTestContextBootstrapper(BootstrapUtils.createBootstrapContext(testClass)));
}
```

这里构建TestContext的逻辑都在`BootstrapUtils`类里面，主要流程：

- 首先使用反射实例化`DefaultCacheAwareContextLoaderDelegate`
- 然后使用反射实例化`DefaultBootstrapContext`
- 然后看看测试类使用是否`@BootstrapWith`注解：
    －如果使用了该注解的话，那么解析出其提供的`TestContextBootstrapper`
    - 如果没使用该注解的话，那么看测试类是否使用`@WebAppConfiguration`注解：
        - 如果使用了`@WebAppConfiguration`注解，那么返回`WebTestContextBootstrapper`
        - 如果没啥用`@WebAppConfiguration`注解，那么返回`DefaultTestContextBootstrapper`

### TestContextBootstrapper

