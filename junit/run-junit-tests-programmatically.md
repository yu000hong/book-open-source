# Running JUnit Tests Programmatically, from a Java Application

In this tutorial, we'll show how to run JUnit tests directly from Java code – there are scenarios where this option comes in handy.

**Maven Dependencies**

```xml
// for JUnit 4
<dependency> 
    <groupId>junit</groupId> 
    <artifactId>junit</artifactId> 
    <version>4.12</version> 
    <scope>test</scope> 
</dependency>
```

**Test Classes**

```java
public class FirstUnitTest {

    @Test
    public void whenThis_thenThat() {
        assertTrue(true);
    }

    @Test
    public void whenSomething_thenSomething() {
        assertTrue(true);
    }

    @Test
    public void whenSomethingElse_thenSomethingElse() {
        assertTrue(true);
    }
}
```

```java
public class SecondUnitTest {

    @Test
    public void whenSomething_thenSomething() {
        assertTrue(true);
    }

    @Test
    public void whensomethingElse_thenSomethingElse() {
        assertTrue(true);
    }
}
```

**Running a Single Test Class**

To run JUnit tests from Java code, we can use the JUnitCore class (with an addition of TextListener class, used to display the output in System.out):

```java
JUnitCore junit = new JUnitCore();
junit.addListener(new TextListener(System.out));
junit.run(FirstUnitTest.class);
```

On the console, we'll see a very simple message indicating successful tests:

```
Running one test class:
..
Time: 0.019
OK (2 tests)
```

**Running Multiple Test Classes**

If we want to specify multiple test classes with JUnit 4, we can use the same code as for a single class, and simply add the additional classes:

```java
JUnitCore junit = new JUnitCore();
junit.addListener(new TextListener(System.out));

Result result = junit.run(
  FirstUnitTest.class, 
  SecondUnitTest.class);

resultReport(result);
```

Note that the result is stored in an instance of JUnit's `Result` class, which we're printing out using a simple utility method:

```java
public static void resultReport(Result result) {
    System.out.println("Finished. Result: Failures: " +
      result.getFailureCount() + ". Ignored: " +
      result.getIgnoreCount() + ". Tests run: " +
      result.getRunCount() + ". Time: " +
      result.getRunTime() + "ms.");
}
```

**Running a Test Suite**

If we need to group some test classes in order to run them, we can create a `TestSuite`. This is just an empty class where we specify all classes using JUnit annotations:

```java
@RunWith(Suite.class)
@Suite.SuiteClasses({
  FirstUnitTest.class,
  SecondUnitTest.class
})
public class MyTestSuite {
}
```

To run these tests, we'll again use the same code as before:

```java
JUnitCore junit = new JUnitCore();
junit.addListener(new TextListener(System.out));
Result result = junit.run(MyTestSuite.class);
resultReport(result);
```

**Running Repeated Tests**

One of the interesting features of JUnit is that we can repeat tests by creating instances of `RepeatedTest`. This can be really helpful when we're testing random values, or for performance checks.

In the next example, we'll run the tests from MergeListsTest five times:

```java
Test test = new JUnit4TestAdapter(FirstUnitTest.class);
RepeatedTest repeatedTest = new RepeatedTest(test, 5);

JUnitCore junit = new JUnitCore();
junit.addListener(new TextListener(System.out));

junit.run(repeatedTest);
```

Here, we're using `JUnit4TestAdapter` as a wrapper for our test class.

We can even create suites programmatically, applying repeated testing:

```java
TestSuite mySuite = new ActiveTestSuite();

JUnitCore junit = new JUnitCore();
junit.addListener(new TextListener(System.out));

mySuite.addTest(new RepeatedTest(new JUnit4TestAdapter(FirstUnitTest.class), 5));
mySuite.addTest(new RepeatedTest(new JUnit4TestAdapter(SecondUnitTest.class), 3));

junit.run(mySuite);
```
