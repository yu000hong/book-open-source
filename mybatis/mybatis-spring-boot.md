# mybatis-spring-boot-starter

我们知道 spring-boot-starter 主要作用有两个：

- 一是提供依赖管理
- 二是提供自动配置

我们先来看看mybatis-spring-boot源码，一共就5个类：

- ConfigurationCustomizer
- MybatisAutoConfiguration
- MybatisLanguageDriverAutoConfiguration
- MybatisProperties
- SpringBootVFS

### spring.factories

```
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.mybatis.spring.boot.autoconfigure.MybatisLanguageDriverAutoConfiguration,\
org.mybatis.spring.boot.autoconfigure.MybatisAutoConfiguration
```

从`spring.factories`文件可以看出，自动配置涉及两个类：

- MybatisLanguageDriverAutoConfiguration
- MybatisAutoConfiguration

### ConfigurationCustomizer

这是一个接口，也是 **Spring Boot** 常用的 Customizer 模式：

```java
@FunctionalInterface
public interface ConfigurationCustomizer {

  void customize(Configuration configuration);

}
```

> **注意⚠️:** 
> - 我们在Spring中可以定义任意多个ConfigurationCustomizer对应的Bean实例，但是最后的执行顺序是不确定的。
> - 如果我们定义了**configLocation**属性，那么任何ConfigurationCustomizer都将无效！

### MybatisProperties

这个类定义了我们在`application.yml`文件中如何设置MyBatis的各种属性，支持的属性有：

- configLocation
- checkConfigLocation
- mapperLocations
- typeAliasesPackage
- typeAliasesSuperType
- typeHandlersPackage
- executorType
- defaultScriptingLanguageDriver
- configurationProperties

`MybatisProperties`只支持这些属性，其他属性的设置可以使用`ConfigurationCustomizer`或者通过`configLocation`指定XML配置文件进行配置。

### SpringBootVFS

从[VFS](vfs.md)我们知道，如果我们不需要遍历包进行mapper/typeAlias/typeHandler的配置的话，其实是可以不需要`SpringBootVFS`。但如果我们配置了使用包注册mapper/typeAlias/typeHandler的话，那么就必须使用`SpringBootVFS`，否则会抛出找不到类资源的异常。

### MybatisAutoConfiguration

先看其构造方法：

```java
public MybatisAutoConfiguration(MybatisProperties properties, ObjectProvider<Interceptor[]> interceptorsProvider,
    ObjectProvider<TypeHandler[]> typeHandlersProvider, ObjectProvider<LanguageDriver[]> languageDriversProvider,
    ResourceLoader resourceLoader, ObjectProvider<DatabaseIdProvider> databaseIdProvider,
    ObjectProvider<List<ConfigurationCustomizer>> configurationCustomizersProvider) {
  this.properties = properties;
  this.interceptors = interceptorsProvider.getIfAvailable();
  this.typeHandlers = typeHandlersProvider.getIfAvailable();
  this.languageDrivers = languageDriversProvider.getIfAvailable();
  this.resourceLoader = resourceLoader;
  this.databaseIdProvider = databaseIdProvider.getIfAvailable();
  this.configurationCustomizers = configurationCustomizersProvider.getIfAvailable();
}
```

从代码可以看出，构造方法里注入了很多类型对应的ObjectProvider，包括：

- Interceptor
- TypeHandler
- LanguageDriver
- DatabaseIdProvider
- ConfigurationCustomizer

因此，我们可以使用`@Component`在代码将这些对象注入到MyBatis!

MybatisAutoConfiguration自动配置了两个类：

- SqlSessionFactory
- SqlSessionTemplate

```java
@Bean
@ConditionalOnMissingBean
public SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception {
  //...
}

@Bean
@ConditionalOnMissingBean
public SqlSessionTemplate sqlSessionTemplate(SqlSessionFactory sqlSessionFactory) {
  //...
}
```

同时，引入了`AutoConfiguredMapperScannerRegistrar`：

```java
@org.springframework.context.annotation.Configuration
@Import(AutoConfiguredMapperScannerRegistrar.class)
@ConditionalOnMissingBean({ MapperFactoryBean.class, MapperScannerConfigurer.class })
public static class MapperScannerRegistrarNotFoundConfiguration implements InitializingBean {

  @Override
  public void afterPropertiesSet() {
    logger.debug(
        "Not found configuration for registering mapper bean using @MapperScan, MapperFactoryBean and MapperScannerConfigurer.");
  }

}
```

`AutoConfiguredMapperScannerRegistrar`这个类的作用就是在环境中没有`MapperScannerConfigurer`对应的实例对象时，自动注册这样一个Bean对象。什么情况下环境中会已经存在`MapperScannerConfigurer`实例Bean对象呢？那就是如果我们使用了`@MapperScan`或者`@MapperScans`注解的时候。

`@MapperScan`/`@MapperScans`这两个注解会通过@Import引入`MapperScannerRegistrar`/`RepeatingRegistrar`，而`MapperScannerRegistrar`/`RepeatingRegistrar`这两个类又会通过`BeanDefinitionBuilder.genericBeanDefinition(MapperScannerConfigurer.class)`自动注册`MapperScannerConfigurer`实例对象。

我们来看看`AutoConfiguredMapperScannerRegistrar.registerBeanDefinitions()`方法：

```java
@Override
public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
  List<String> packages = AutoConfigurationPackages.get(this.beanFactory);
  BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition(MapperScannerConfigurer.class);
  builder.addPropertyValue("processPropertyPlaceHolders", true);
  builder.addPropertyValue("annotationClass", Mapper.class);
  builder.addPropertyValue("basePackage", StringUtils.collectionToCommaDelimitedString(packages));
  BeanWrapper beanWrapper = new BeanWrapperImpl(MapperScannerConfigurer.class);
  Stream.of(beanWrapper.getPropertyDescriptors())
      // Need to mybatis-spring 2.0.2+
      .filter(x -> x.getName().equals("lazyInitialization")).findAny()
      .ifPresent(x -> builder.addPropertyValue("lazyInitialization", "${mybatis.lazy-initialization:false}"));
  registry.registerBeanDefinition(MapperScannerConfigurer.class.getName(), builder.getBeanDefinition());
}
```

它会使用`AutoConfigurationPackages`发现所有自动配置注册的包，并在这些包下面搜索带有`@Mapper`注解的映射器接口。

如果使用`@MapperScan/@MapperScans`，我们可以通过组合annotationClass和markerInterface来进行搜索过滤：

- annotationClass: 指定接口必须带有对应注解
- markerInterface: 指定接口必须继承自对应类

> **问题**：如果我们同时使用`mapperLocations`属性和`MapperScannerConfigurer`指定了相同的包，会报错么？

> **回答**：这里理解出现了偏差，`mapperLocations`是为了设置MyBatis对应的映射器，而`MapperScannerConfigurer`是为了生成Mapper对应的实例Bean对象，也即是每个Mapper接口会生成一个对应的MapperFactoryBean对象。
>
> 如果我们没有设置`mapperLocations`，只要`MapperScannerConfigurer`能扫描到我们所有的Mapper接口并且MapperFactoryBean的addToConfig属性设置为true，那么`MapperScannerConfigurer`会帮我们完成MyBatis里面mapper的注册。

看看MapperFactoryBean源码：

```java
@Override
protected void checkDaoConfig() {
  super.checkDaoConfig();
  Configuration configuration = getSqlSession().getConfiguration();
  if (this.addToConfig && !configuration.hasMapper(this.mapperInterface)) {
    try {
      // 这里会帮助我们将mapper自动注册到MyBatis
      configuration.addMapper(this.mapperInterface);
    } catch (Exception e) {
      logger.error("Error while adding the mapper '" + this.mapperInterface + "' to configuration.", e);
      throw new IllegalArgumentException(e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
}
```