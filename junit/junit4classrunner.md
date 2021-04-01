# JUnit4ClassRunner


`JUnit4ClassRunner`虽然已经被废弃了，但是它代码简单，我们可以先从简单的入手，看看Runner的执行逻辑。

几个相关类：

- TestClass(即将废弃)
- TestMethod(即将废弃)
- MethodValidator(即将废弃)
- ClassRoadie(即将废弃)
- MethodRoadie(即将废弃)

### TestClass

`TestClass`的作用是解析测试类，提供一些元数据，包括：

- 所有带有`@Test`注解的方法
- 所有带有`@BeforeClass`注解的方法
- 所有带有`@AfterClass`注解的方法
- 所有带有`@Before`注解的方法
- 所有带有`@After`注解的方法

同时，如果测试类带有`@FixMethodOrder`注解，那么这些方法将会按照注解指定的排序比较器进行排序。

### TestMehtod

`TestMethod`的作用是解析测试方法，提供一些元数据，包括：

- 是否包含`@Ignore`注解
- 返回`@Test`注解上的timeout超时时间参数
- 返回`@Test`注解上的expected期望异常参数
- 通过`TestClass`返回所有带有`@Before`注解的方法
- 通过`TestClass`返回所有带有`@After`注解的方法

### MethodValidator

执行测试类上方法的校验，包括：

- 测试类必须包含无参的默认构造方法，同时必须是public公共类
- `@After`注解的方法不能是静态方法
- `@Before`注解的方法不能是静态方法
- `@Test`注解的方法不能是静态方法
- 测试类中至少要包含一个带有`@Test`注解的方法
- `@AfterClass`注解的方法必须是静态方法
- `@BeforeClass`注解的方法必须是静态方法
- 所有这些带有注解的方法必须满足如下条件：
    - 必须返回void类型
    - 必须是公共方法
    - 必须是无参方法

### ClassRoadie

`ClassRoadie`的主要方法是`runProtected()`：

```java
public void runProtected() {
    try {
        runBefores();
        runUnprotected();
    } catch (FailedBefore e) {
    } finally {
        runAfters();
    }
}
```

- `runBefores()`：按序执行测试类中所有带有`@BeforeClass`注解的方法
- `runUnprotected()`：按序执行`JUnit4ClassRunner.runMethods()`方法，下面讲
- `runAfters()`：按序执行测试类中所有带有`@AfterClass`注解的方法

### MethodRoadie

`MethodRoadie`的主要方法：

- `runBefores()`：依次执行带有`@Before`注解的方法
- `runAfters()`：依次执行带有`@After`注解的方法
- `runTestMethod()`：执行测试方法，并判断是否有满足期望的异常抛出
- `runWithTimeout()`：开启一个只有一个线程的线程池独立执行测试方法，超时则强制关闭线程池

主要逻辑`run()`：

```java
public void run() {
    if (testMethod.isIgnored()) {
        notifier.fireTestIgnored(description);
        return;
    }
    notifier.fireTestStarted(description);
    try {
        long timeout = testMethod.getTimeout();
        if (timeout > 0) {
            runWithTimeout(timeout);
        } else {
            runTest();
        }
    } finally {
        notifier.fireTestFinished(description);
    }
}
```

```java
private void runWithTimeout(final long timeout) {
    runBeforesThenTestThenAfters(new Runnable() {
        public void run() {
            ExecutorService service = Executors.newSingleThreadExecutor();
            Callable<Object> callable = new Callable<Object>() {
                public Object call() throws Exception {
                    runTestMethod();
                    return null;
                }
            };
            Future<Object> result = service.submit(callable);
            service.shutdown();
            try {
                boolean terminated = service.awaitTermination(timeout,
                        TimeUnit.MILLISECONDS);
                if (!terminated) {
                    service.shutdownNow();
                }
                result.get(0, TimeUnit.MILLISECONDS); // throws the exception if one occurred during the invocation
            } catch (TimeoutException e) {
                addFailure(new TestTimedOutException(timeout, TimeUnit.MILLISECONDS));
            } catch (Exception e) {
                addFailure(e);
            }
        }
    });
}
```

```java
public void runTest() {
    runBeforesThenTestThenAfters(new Runnable() {
        public void run() {
            runTestMethod();
        }
    });
}
```

```java
protected void runTestMethod() {
    try {
        testMethod.invoke(test);
        if (testMethod.expectsException()) {
            addFailure(new AssertionError("Expected exception: " + testMethod.getExpectedException().getName()));
        }
    } catch (InvocationTargetException e) {
        Throwable actual = e.getTargetException();
        if (actual instanceof AssumptionViolatedException) {
            return;
        } else if (!testMethod.expectsException()) {
            addFailure(actual);
        } else if (testMethod.isUnexpected(actual)) {
            String message = "Unexpected exception, expected<" + testMethod.getExpectedException().getName() + "> but was<"
                    + actual.getClass().getName() + ">";
            addFailure(new Exception(message, actual));
        }
    } catch (Throwable e) {
        addFailure(e);
    }
}
```

```java
public void runBeforesThenTestThenAfters(Runnable test) {
    try {
        runBefores();
        test.run();
    } catch (FailedBefore e) {
    } catch (Exception e) {
        throw new RuntimeException("test should never throw an exception to this level");
    } finally {
        runAfters();
    }
}
```

### JUnit4ClassRunner

`JUnit4ClassRunner`的核心逻辑就是：

- 使用`MethodValidator`对测试类进行校验
- 使用`ClassRoadie`执行`@BeforeClass`和`@AfterClass`方法
- 使用`MethodRoadie`执行单元测试方法逻辑，即：`@Before`->`@Test`->`@After` 

`JUnit4ClassRunner`核心逻辑就是上面我们提到的`runMethods()`方法：

```java
protected void runMethods(final RunNotifier notifier) {
    for (Method method : testMethods) {
        invokeTestMethod(method, notifier);
    }
}
```

```java
protected void invokeTestMethod(Method method, RunNotifier notifier) {
    Description description = methodDescription(method);
    Object test;
    try {
        test = createTest();
    } catch (InvocationTargetException e) {
        testAborted(notifier, description, e.getCause());
        return;
    } catch (Exception e) {
        testAborted(notifier, description, e);
        return;
    }
    TestMethod testMethod = wrapMethod(method);
    new MethodRoadie(test, testMethod, notifier, description).run();
}
```

从`invokeTestMethod()`方法我们可以看出，每个测试方法的执行都会重新实例化一个测试类的对象！

