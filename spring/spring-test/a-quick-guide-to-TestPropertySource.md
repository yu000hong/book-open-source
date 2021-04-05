# @TestPropertySource 简介

当我们在进行测试的时候，需要使用一些测试属性来提供一些测试场景或者覆盖项目中的属性的时候，我们就可以使用`@TestPropertySource`注解。`@TestPropertySource`注解定义的属性具有更高的优先级，因此可以直接覆盖项目中的属性。

```java
@Component
public class ClassUsingProperty {
    
    @Value("${testpropertysource.one}")
    private String propertyOne;
    
    public String retrievePropertyOne() {
        return propertyOne;
    }
}
```

```java
@RunWith(SpringRunner.class)
@ContextConfiguration(classes = ClassUsingProperty.class)
@TestPropertySource
public class DefaultTest {
    @Autowired
    ClassUsingProperty classUsingProperty;

    @Test
    public void givenDefaultTPS_whenVariableRetrieved_thenDefaultFileReturned() {
        String output = classUsingProperty.retrievePropertyOne();
        assertThat(output).isEqualTo("default-value");
    }
}
```

> Typically, whenever we use this test annotation we will also include the `@ContextConfiguration` so as to load and configure the **ApplicationContext** for the scenario.

默认情况下，`@TestPropertySource`会根据其注解的测试类去查找对应的属性文件。这里假设类DefaultTest位于**org.test**包下，那么就会去类路径下查找**org/test/DefaultTest.properties**这个属性文件。

当然，我们也可以显示的指定属性文件地址：

```java
@RunWith(SpringRunner.class)
@ContextConfiguration(classes = ClassUsingProperty.class)
@TestPropertySource(locations = "/other-location.properties")
public class DefaultTest {
    //...
}
```

甚至，我们可以直接在`@TestPropertySource`里面直接指定属性值：

```java
@RunWith(SpringRunner.class)
@ContextConfiguration(classes = ClassUsingProperty.class)
@TestPropertySource(locations = "/other-location.properties",
    properties = "testpropertysource.one=other-property-value")
public class DefaultTest {
    //...
}
```
