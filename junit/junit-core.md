# 执行核心 - JUnitCore

> The test cases are executed using `JUnitCore` class. JUnitCore is a facade for running tests. It supports running JUnit 4 tests, JUnit 3.8.x tests, and mixtures. 

`JUnitCore`是JUnit的执行核心，所有测试执行逻辑都从这里开始，我们可以使用三种方式进行测试：

- 命令行执行
- 一次性执行
- 定制化执行

### 命令行执行

我们可以直接从命令行中开始执行测试：

```bash
java org.junit.runner.JUnitCore TestClass1 TestClass2 TestSuit3...
```

我们来看看main方法：

```java
public static void main(String... args) {
    Result result = new JUnitCore().runMain(new RealSystem(), args);
    System.exit(result.wasSuccessful() ? 0 : 1);
}
```

### 一次性执行

`JUnitCore`提供了静态方法`runClasses()`，我们可以通过它进行一次性执行：

```java
public static void main(String[] args) {
    Result result = JUnitCore.runClasses(CalculatorTest.class);
    for(Failure failure : result.getFailures()) {
            System.out.print(failure.getTestHeader() +
                "： " + failure.getMessage());
    }
}
```

静态方法`runClasses()`也是实例化`JUnitCore`，然后执行`run()`方法：

```java
public static Result runClasses(Computer computer, Class<?>... classes) {
    return new JUnitCore().run(computer, classes);
}
```

### 定制化执行

通过实例化`JUnitCore`创建对应的实例，然后调用其方法进行定制化，最后执行`run()`方法执行测试：

```java
public static void main(String[] args) {
    JUnitCore junitCore = new JUnitCore();
    junitCore.addListener(new RunListener() {
        //一些感兴趣的方法实作
        public void testStarted(Description description) throws Exception {
            System.out.println(description.getDisplayName() + "...started");
        }
    });
    Result result = junitCore.run(CalculatorTest.class);
    for(Failure failure : result.getFailures()) {
        System.out.print(failure.getTestHeader() +
                 "： " + failure.getMessage());
    }
}
```

### 核心方法

`JUnitCore`的核心方法是`run()`，所有逻辑最后都会走到这个方法：

```java
public Result run(Runner runner) {
    Result result = new Result();
    RunListener listener = result.createListener();
    notifier.addFirstListener(listener);
    try {
        notifier.fireTestRunStarted(runner.getDescription());
        runner.run(notifier);
        notifier.fireTestRunFinished(result);
    } finally {
        removeListener(listener);
    }
    return result;
}
```

这里的关键部分还是如何根据前面的`Class<?>... classes`测试类参数确定我们最终要使用具体哪个`Runner`。

我们先看几个重要的类：

- Suite
- Computer
- Request
- RunnerBuilder
- RunnerScheduler
- Runner
- @RunWith

**Runner**

每一个测试类最终都会对应一个Runner，Runner是抽象类，有层级关系。

```
Runner
  |---ErrorReportingRunner(内部使用)
  |---JUnit4ClassRunner
  |---JUnit38ClassRunner
  |---IgnoredClassRunner
  |---RunnerSpy
  |---ParentRunner
         |---BlockJUnit4ClassRunner
         |     |---BlockJUnit4ClassRunnerWithParameters
         |     |---Theories
         |     |---JUnit4
         |
         |---Suite
            |---Enclosed
            |---Categories
            |---Parameterized
```

**Suite**

首先，我们所有的测试类最终都会形成一个测试套件Suite，Suite本身就是一个Runner！

**Computer**

> Represents a strategy for computing runners and suites.

`Computer`表示的执行runner和suite的一种策略，默认是串行执行的。其子类`ParallelComputer`表示的是并行执行策略，它使用了一个线程池来并行执行测试单元。

**Request**

> A `Request` is an abstract description of tests to be run. Older versions of
> JUnit did not need such a concept--tests to be run were described either by classes 
> containing tests or a tree of `org.junit.Test`s. However, we want to support filtering and
> sorting, so we need a more abstract specification than the tests themselves and a richer
> specification than just the classes.

`Request`是一个抽象类，有一个抽象方法`getRunner()`。Request提供了对测试方法集进行过滤和排序的能力，具体的方法就是：

```java
public Request filterWith(Filter filter) {
    return new FilterRequest(this, filter);
}
public Request filterWith(Description desiredDescription) {
    return filterWith(Filter.matchMethodDescription(desiredDescription));
}
public Request sortWith(Comparator<Description> comparator) {
    return new SortingRequest(this, comparator);
}
public Request orderWith(Ordering ordering) {
    return new OrderingRequest(this, ordering);
}
```

继承关系：

```
Request
  |---SortingRequest
  |---FilterRequest
  |---MemoizingRequest(内部使用)
        |---OrderingRequest
        |---ClassRequest
```

最重要的一个方法：

```java
public static Request classes(Computer computer, Class<?>... classes) {
    try {
        AllDefaultPossibilitiesBuilder builder = new AllDefaultPossibilitiesBuilder();
        Runner suite = computer.getSuite(builder, classes);
        return runner(suite);
    } catch (InitializationError e) {
        return runner(new ErrorReportingRunner(e, classes));
    }
}
```

这个方法就是将我们提供的测试类转换成对应的Request对象，从代码可以看出，我们提供的测试类最终要打包成一个测试套件Suite。我们再来看看我们的测试类最终对应的Runner是怎么确定的，看看`AllDefaultPossibilitiesBuilder`的方法：

```java
public Runner runnerForClass(Class<?> testClass) throws Throwable {
    List<RunnerBuilder> builders = Arrays.asList(
            ignoredBuilder(),
            annotatedBuilder(),
            suiteMethodBuilder(),
            junit3Builder(),
            junit4Builder());

    for (RunnerBuilder each : builders) {
        Runner runner = each.safeRunnerForClass(testClass);
        if (runner != null) {
            return runner;
        }
    }
    return null;
}
```

`AllDefaultPossibilitiesBuilder`继承自`RunnerBuilder`，它通过自己的判断规则来发现我们的测试类最终需要在什么Runner上执行：

- 首先判断测试类是否有`@Ignore`注解，如果有，使用`IgnoredClassRunner`；
- 然后判断测试类是否有`@RunWith`注解，如果有，使用其提供的类并进行实例化；
- 然后判断测试类是否包含`suite()`方法，如果有，使用`SuiteMehtod`；
- 然后判断测试类是否继承自`junit.framework.TestCase`，如果是，使用`JUnit38CLassRunner`；
- 否则，就使用`JUnit4`，也即`BlockJUnit4ClassRunner`。

