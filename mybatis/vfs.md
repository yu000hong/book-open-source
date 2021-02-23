# VFS

VFS的作用是遍历某个包(package)下的所有文件，主要用于遍历查找包下这三类文件：

- **映射器**: mapper class
- **类型别名**: type alias
- **类型处理器**: type handler

### 抽象方法

VFS主要抽象方法：

```java
List<String> list(URL url, String path)
```

其中，path为包的路径名形式，如`com.yu000hong.mapper`对应的值为：**com/yu000hong/mapper**。

我们需要实现的就是这个list方法，根据包路径名和对应资源URL来遍历并返回所有包下的文件，包括各种资源文件及类文件。

MyBatis内置了两个实现：

- DefaultVFS：TODO
- JBoss6VFS：TODO

### ResolverUtil

`VFS`的使用只有一个地方，那就是`ResolverUtil`，这个类的作用就是遍历指定的package，然后筛选出我们需要的类型。`ResolverUtil`的使用涉及到如下三个类：

- MapperRegistry
- TypeAliasRegistry
- TypeHandlerRegistry

也就是说，我们在注册mapper/typeAlias/typeHandler的时候就会用到VFS的功能。这里要区分一下，如果直接注册指定的类是不需要VFS的，直接使用ClassLoader加载类就行了；如果注册指定的包的话，就需要VFS去遍历了。

这也是为什么**spring-boot-loader**模块已经提供了`LaunchedURLClassLoader`用于加载FatJar嵌套的Jar文件，**mybatis-spring-boot**还需要提供`SpringBootVFS`这样一个特定的VFS实现。
