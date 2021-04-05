# ParentRunner

`ParentRunner`提供了一种层次化的结构，使我们的测试形成一个树形结构，目前有两种实现方式：

- Suite：将Runner组织成层次化结构
- BlockJUnit4ClassRunner：将测试方法组织成层次化结构

> Subclasses must implement finding the children of the node, describing each child, and running each child. ParentRunner will filter and sort children, handle `@BeforeClass` and `@AfterClass` methods, handle annotated `ClassRule`s, create a composite `Description`, and run children sequentially.

抽象方法：

- protected abstract List<T> getChildren();
- protected abstract Description describeChild(T child);
- protected abstract void runChild(T child, RunNotifier notifier);

相关类：

- FrameworkMember
- FrameworkField
- FrameworkMethod
- TestClass
- TestClassValidator
- Statement

### FrameworkMember

> FrameworkMember is parent class for `FrameworkField` and `FrameworkMethod`

### FrameworkField

`FrameworkField`用于表示测试类中的一个字段，目前仅在`BlockJUnit4ClassRunner`用于处理规则。自定义Runner实现类可以充分使用`FrameworkField`。

### FrameworkMethod

`FrameworkMethod`用于表示测试类中的一个方法，这些方法通常都带有`@Test`/`@Before`/`@After`/`@BeforeClass`/`@AfterClass`这些注解。

### TestClass

`TestClass`是测试类的包装类，用于提供方法校验和注解搜索功能。`TestClass`根据测试类的类元信息扫描所有带有注解的字段和方法，生成相应的**注解->字段**映射和**注解->方法**映射。

字段会根据名称字符串的比较进行排序，方法的排序分为两种情况：

- 如果测试类带有`@FixMethodOrder`注解，那么根据注解提供的排序比较器进行排序
- 如果没有这个注解，那么使用默认的排序器进行排序（根据方法名的哈希排序）

### TestClassValidator

`TestClassValidator`是一个接口，用于校验测试类的，接口方法：

```java
List<Exception> validateTestClass(TestClass testClass);
```

如果校验过程发现了异常，那么将异常以列表的形式返回；否则，返回空列表。

实现类有：

- PublicClassValidator：判断测试类是否是公共类
- AnnotationsValidator：使用测试类`@ValidateWith`注解提供的校验器进行校验

### Statement

`Statement`表示一个具体的执行单元，可以通过内部的next变量把多个执行单元进行串行化执行。它是一个抽象类，代码如下：

```java
public abstract class Statement {
    public abstract void evaluate() throws Throwable;
}
```

`Statement`有很多实现类：

- RunBefores：依次执行所有`@Before`注解标注的方法
- RunAfters：依次执行所有`@After`注解标注的方法
- RunRules：依次执行所有`TestRule`规则
- InvokeMenthod：执行具体的测试方法
- ExpectException：执行测试方法，判断是否有预期异常抛出
- FailOnTimeout：单独开启一个线程来执行，并且使用了`ThreadGroup`(为啥使用线程组，这里单个线程如何终止的？TODO)
- Fail：直接抛出异常

### ParentRunner

`ParentRunner`在构造方法中会进行校验，校验内容包括：

- `@AfterClass`注解的方法必须是静态公共无参方法
- `@BeforeClass`注解的方法必须是静态公共无参方法
- 测试类必须是公共类
- `@ClassRule`注解的字段必须是静态公共字段，并且字段类型必须是`TestRule`
- `@ClassRule`注解的方法必须是静态公共方法，并且返回类型必须是`TestRule`
- 根据**VALIDATORS**中指定的校验器进行校验，**VALIDATORS**中的校验器由子类提供

看一下`run()`方法代码：

```java
public void run(final RunNotifier notifier) {
    EachTestNotifier testNotifier = new EachTestNotifier(notifier,
            getDescription());
    testNotifier.fireTestSuiteStarted();
    try {
        Statement statement = classBlock(notifier);
        statement.evaluate();
    } catch (AssumptionViolatedException e) {
        testNotifier.addFailedAssumption(e);
    } catch (StoppedByUserException e) {
        throw e;
    } catch (Throwable e) {
        testNotifier.addFailure(e);
    } finally {
        testNotifier.fireTestSuiteFinished();
    }
}
```

从代码可以看出，所有逻辑都在`classBlock()`方法里面。

```java
protected Statement classBlock(final RunNotifier notifier) {
    Statement statement = childrenInvoker(notifier);//先执行子节点逻辑
    if (!areAllChildrenIgnored()) {
        statement = withBeforeClasses(statement);
        statement = withAfterClasses(statement);
        statement = withClassRules(statement);
        statement = withInterruptIsolation(statement);
    }
    return statement;
}
protected Statement childrenInvoker(final RunNotifier notifier) {
    return new Statement() {
        @Override
        public void evaluate() {
            runChildren(notifier);
        }
    };
}
private void runChildren(final RunNotifier notifier) {
    final RunnerScheduler currentScheduler = scheduler;//默认是串行执行
    try {
        for (final T each : getFilteredChildren()) {
            currentScheduler.schedule(new Runnable() {
                public void run() {
                    ParentRunner.this.runChild(each, notifier);
                }
            });
        }
    } finally {
        currentScheduler.finished();
    }
}
```

### Suite

> Using `Suite` as a runner allows you to manually build a suite containing tests from many classes. It is the JUnit 4 equivalent of the JUnit 3.8.x static `junit.framework.Test.suite()` method. To use it, annotate a class with `@RunWith(Suite.class)` and `@SuiteClasses({TestClass1.class, ...})`. When you run this class, it will run all the tests in all the suite classes.

我们在使用JUnitCore的时候，也会把我们提供的测试类自动封装到一个Suite里面进行执行。

我们在使用Suite的时候，一定要同时在对应的测试类上使用`@SuiteClasses`注解，如下：

```java
@RunWith(Suite.class)
@SuiteClasses({Test1.class, Test2.class})
public class MyTestSuite {

}
```

具体实现方法很简单：

```java
@Override
protected List<Runner> getChildren() {
    return runners;
}
@Override
protected Description describeChild(Runner child) {
    return child.getDescription();
}
@Override
protected void runChild(Runner runner, final RunNotifier notifier) {
    runner.run(notifier);//直接使用子节点Runner进行执行
}
```

### BlockJUnit4ClassRunner

`BlockJUnit4ClassRunner`在父类`ParentRunner`基础上增加了如下校验：

- 测试类必须具有唯一一个公共无参构造方法
- 测试类不能具有内部类
- `@After`注解的方法必须是非静态公共无参方法，同时返回void类型
- `@Before`注解的方法必须是非静态公共无参方法，同时返回void类型
- 测试类必须具有至少一个测试方法（由`@Test`注解）
- 所有测试方法必须是非静态公共无参方法，同时返回void类型
- `@Rule`注解的字段必须是非静态的公共字段，同时类型必须是`TestRule`或者`MethodRule`
- `@Rule`注解的方法必须是非静态的公共方法，同时返回`TestRule`或者`MethodRule`类型

核心方法：

```java
protected void runChild(final FrameworkMethod method, RunNotifier notifier) {
    Description description = describeChild(method);
    if (isIgnored(method)) {
        notifier.fireTestIgnored(description);
    } else {
        Statement statement = new Statement() {
            @Override
            public void evaluate() throws Throwable {
                methodBlock(method).evaluate();
            }
        };
        runLeaf(statement, description, notifier);
    }
}
```

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
    statement = withRules(method, test, statement);
    statement = withInterruptIsolation(statement);
    return statement;
}
```

```java
protected Statement methodInvoker(FrameworkMethod method, Object test) {
    return new InvokeMethod(method, test);
}
```

所有执行逻辑都是通过`Statement`及其子类来完成的，`BlockJUnit4ClassRunner`的整体执行逻辑为：

TODO



