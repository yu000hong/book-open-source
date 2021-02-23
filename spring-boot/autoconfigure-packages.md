# @AutoConfigurationPackage 

> Indicates that the package containing the annotated class should be registered with AutoConfigurationPackages.

包含`@AutoConfigurationPackage`这个注解的package，需要使用`AutoConfigurationPackages`将package进行注册，后续会使用到这些包。MyBatis里面自动映射器发现就使用到了`AutoConfigurationPackages`里面注册到的包名，根据这些包名去扫描里面带有`@Mapper`注解的映射器类。

### EnableAutoConfiguration

我们看看其源码：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {

	String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

	Class<?>[] exclude() default {};

	String[] excludeName() default {};

}
```

可以看出，`@AutoConfigurationPackage`是其元注解，因此当我们在使用`@EnableAutoConfiguration`注解的时候，就已经将对应的包名注册到`AutoConfigurationPackages`实例对象里面了。