# SpringJUnit4ClassRunner

spring-test项目与JUnit相关的类都在`org.springframework.test.context.junit4`包下，主要有：

- SpringJUnit4ClassRunner
- SpringRunner
- SpringClassRule
- SpringMethodRule
- AbstractJUnit4SpringContextTests
- AbstractTransactionalJUnit4SpringContextTests

以及一些控制执行方式的Statements：

- SpringRepeat
- SpringFailOnTimeout
- ProfileValueChecker
- RunAfterTestClassCallbacks
- RunAfterTestExecutionCallbacks
- RunAfterTestMethodCallbacks
- RunBeforeTestClassCallbacks
- RunBeforeTestExecutionCallbacks
- RunBeforeTestMethodCallbacks
- RunPrepareTestInstanceCallbacks

### SpringJUnit4ClassRunner

`SpringJUnit4ClassRunner`继承自`BlockJUnit4ClassRunner`，提供了Spring TestContext框架，用于提供支持Spring的测试能力。

首先看看构造方法：

```java
public SpringJUnit4ClassRunner(Class<?> clazz) throws InitializationError {
    super(clazz);
    if (logger.isDebugEnabled()) {
        logger.debug("SpringJUnit4ClassRunner constructor called with [" + clazz + "]");
    }
    ensureSpringRulesAreNotPresent(clazz);
    this.testContextManager = createTestContextManager(clazz);
}
```

构造方法主要逻辑：

- 确保测试类里的字段不会具有`SpringClassRule`或`SpringMethodRule`，因为这两种规则不能和`SpringJUnit4ClassRunner`一起使用。我们使用其他Runner的时候，为了能够使用Spring相关的功能，就可以直接使用这两种规则。
- 创建`TestContextManager`。

再来看核心`run()`方法：

```java
public void run(RunNotifier notifier) {
    if (!ProfileValueUtils.isTestEnabledInThisEnvironment(getTestClass().getJavaClass())) {
        notifier.fireTestIgnored(getDescription());
        return;
    }
    super.run(notifier);
}
```

如果使用了`@IfProfileValue`，那么我们需要判断测试类是否所在的测试分组是否启用，如果启用直接使用父类的使用执行测试逻辑。
虽然使用的是父类的测试逻辑，但是`SpringJUnit4ClassRunner`重写了不少父类的方法来实现Spring相关功能，重写方法有：

- createTest
- runChild
- withBeforeClasses
- withAfterClasses
- withBefores
- withAfters
- methodBlock
- possiblyExpectingExceptions
- withPotentialTimeout

### createTest

```java
@Override
protected Object createTest() throws Exception {
    Object testInstance = super.createTest();
    getTestContextManager().prepareTestInstance(testInstance);
    return testInstance;
}
```










