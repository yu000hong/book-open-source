# JUnit Rules Model

我们一般在写单元测试的时候，会做一些初始化和清理工作。针对整个测试类的初始化工作使用`@BeforeClass`，清理工作使用`@AfterClass`，他们必须作用在静态方法上。针对单个测试方法的初始化工作使用`@Before`，清理工作使用`@After`，他们作用在实例方法上。

相较于`@BeforeClass`/`@AfterClass`/`@Before`/`@After`，**JUnit规则模型**提供了更加灵活强大的机制来做一些测试准备清理工作，不仅支持**before**/**after**语义，甚至可以支持**around**语义，而且你可以自己实现特殊的逻辑。

### @ClassRule

`@ClassRule`可以注解字段或者方法，其注解的字段必须是静态公共字段且字段类型必须是`TestRule`，其注解的方法必须是静态公共方法且返回类型必须是`TestRule`。也就是说，不论注解字段还是方法，最终我们要获得的都是一个`TestRule`对象。

`@ClassRule`注解的字段或者方法对应的`TestRule`只会执行一次，具体代码在`ParentRunner.classBlock()`里面：

```java
protected Statement classBlock(final RunNotifier notifier) {
    Statement statement = childrenInvoker(notifier);//先执行子节点逻辑
    if (!areAllChildrenIgnored()) {
        statement = withBeforeClasses(statement);
        statement = withAfterClasses(statement);
        statement = withClassRules(statement);//执行@ClassRule逻辑
        statement = withInterruptIsolation(statement);
    }
    return statement;
}
```

### @Rule

`@Rule`可以注解字段或者方法，其注解的字段必须是非静态的公共字段且类型必须是`TestRule`或者`MethodRule`，其注解的方法必须是非静态的公共方法且返回`TestRule`或者`MethodRule`类型。也就是说，不论注解字段还是方法，最终我们要获得的是一个`TestRule`或`MethodRule`对象。

`@Rule`注解的字段或者方法对应的`TestRule`或`MethodRule`会为每个测试方法执行一次，具体代码在`BlockJUnit4ClassRunner.methodBlock()`里面：

```java
protected Statement methodBlock(final FrameworkMethod method) {
    Object test;
    try {
        test = new ReflectiveCallable() {
            @Override
            protected Object runReflectiveCall() throws Throwable {
                return createTest(method);
            }
        }.run();
    } catch (Throwable e) {
        return new Fail(e);
    }

    Statement statement = methodInvoker(method, test);
    statement = possiblyExpectingExceptions(method, test, statement);
    statement = withPotentialTimeout(method, test, statement);
    statement = withBefores(method, test, statement);
    statement = withAfters(method, test, statement);
    statement = withRules(method, test, statement);//执行@Rule逻辑
    statement = withInterruptIsolation(statement);
    return statement;
}
```

### TestRule

`TestRule`是一个接口，代码：

```java
public interface TestRule {
    Statement apply(Statement base, Description description);
}
```

我们可以在`apply()`里面加入我们的执行逻辑，比如一些初始化工作、一些清理工作等，如：

```java
public class MyBeforeRule {
    public Statement apply(Statement base, Description description){
        init();
        base.evaluate();
    }
}
public class MyAfterRule {
    public Statement apply(Statement base, Description description){
        base.evaluate();
        clean();
    }
}
public class MyAroundRule {
    public Statement apply(Statement base, Description description){
        init();
        base.evaluate();
        clean();
    }
}
public class MyIgnoreRule {
    public Statement apply(Statement base, Description description){
        doSomething();
    }
}
```

我们这里实现了**before**、**after**、**around**语义，甚至`MyIgnoreRule`直接忽略了原测试逻辑的执行。

通过`TestRule`你可以实现任何你想要的执行逻辑，所以我们说`JUnit规则模型`为我们提供了更灵活更强大的辅助能力。

### MethodRule

`MethodRule`是一个接口，代码：

```java
public interface MethodRule {
    Statement apply(Statement base, FrameworkMethod method, Object target);
}
```

> **注意**：`MethodRule`已经被废弃，建议使用`TestRule`进行替换！

**为什么`MethodRule`会被废弃呢？**

原因很简单，`MethodRule`接口违背了一些设计准则，将JUnit内部使用的类`FrameworkMethod`暴露了出来，所以最后被`TestRule`替换了！

