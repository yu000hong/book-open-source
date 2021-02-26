# Spring 常用问题总结

### 我们什么时候用@Component注解，什么时候用@Bean注解？

Spring的配置主要有三种方式：

- XML方式
- 注解方法
- JavaConfig方式

XML方式就不说了，现在基本很少用了。注解方式指的就是使用`@Component`注解及其子注解来定义实例Bean。JavaConfig方式指的就是使用`@Configuration`和`@Bean`注解来定义实例Bean。

> **注意⚠️**: 注解没有继承关系，所谓的@Component子注解说的就是那些使用了@Component作为其元注解的注解，比如：@Service、@Repository、@Controller等。

那么什么时候使用注解方式，什么时候使用JavaConfig方式呢？

答案就是如果是我们应用内的类的话，直接使用`@Component`就可定义Bean实例了；但是，对于外部系统引入的类，我们没法在其类上加注解，这种情况就必须使用XML方式(还是提一下)或者JavaConfig方式，通过代码的形式(定义一个带`@Bean`注解的方法)来引入外部系统第三方依赖。