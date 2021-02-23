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

我们在Spring中可以定义任意多个ConfigurationCustomizer对应的Bean实例，但是最后的执行顺序是不确定的。

### MybatisProperties

这各类定义了我们在`application.yml`文件中如何设置MyBatis的各种属性，支持的属性有：

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