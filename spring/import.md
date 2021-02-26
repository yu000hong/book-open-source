# @Import

> Indicates one or more component classes to import. Allows for importing `@Configuration` classes, `ImportSelector` and `ImportBeanDefinitionRegistrar` implementations, as well as regular component classes. 

从官方文档可以看出，`@Import`支持导入四种类型：

- `@Configuration` class
- `ImportSelector` implementation
- `ImportBeanDefinitionRegistrar` implementation
- regular component class

### ImportSelector

### ImportBeanDefinitionRegistrar

### @Import vs @ComponentScan

> **相似处**: Both annotations can accept any @Component or @Configuration class.

> **不同处**: To answer this question, let's remember that Spring generally promotes the convention-over-configuration approach. 
>
> Making an analogy with our annotations, @ComponentScan is more like convention, while @Import looks like configuration.

Typically, we start our applications using @ComponentScan in a root package so it can find all components for us. If we're using Spring Boot, then @SpringBootApplication already includes @ComponentScan, and we're good to go. This shows the power of convention.

Now, let's imagine that our application is growing a lot. Now we need to deal with beans from all different places, like components, different package structures, and modules built by ourselves and third parties. 

In this case, adding everything into the context risks starting conflicts about which bean to use. Besides that, we may get a slow start-up time.

On the other hand, we don't want to write an @Import for each new component because doing so is counterproductive.

**We can aim for the best of both worlds.** Let's picture that we have a package only for our animals. It could also be a component or module and keep the same idea. Then we can have one @ComponentScan just for our animal package:

```java
package com.baeldung.importannotation.animal;

@Configuration
@ComponentScan
public class AnimalScanConfiguration {
}
```

And an @Import to keep control over what we'll add to the context:

```java
package com.baeldung.importannotation.zoo;

@Configuration
@Import(AnimalScanConfiguration.class)
class ZooApplication {
}
```

Finally, any new bean added to the animal package will be automatically found by our context. And we still have explicit control over the configurations we are using.

