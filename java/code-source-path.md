# 如何获取Java某个类所在源文件的位置？

目前知道的有两种方式：

- `Class.getResource("")`
- `Class.getProtectionDomain().getCodeSource()`

```java
public class Main {

    public static void main(String[] args){
        printResourcePath(Main.class);
        printCodeSource(Main.class);

        printResourcePath(Mapper.class);
        printCodeSource(Mapper.class);

        printResourcePath(String.class);
        printCodeSource(String.class);
    }
    
    private static void printResourcePath(Class<?> clz){
        System.out.println("=====resource path=====");
        String path = clz.getResource("").getPath();
        System.out.println(path);
    }
    
    private static void printCodeSource(Class<?> clz){
        System.out.println("=====code source=====");
        String path = clz.getProtectionDomain().getCodeSource().getLocation().getPath();
        System.out.println(path);
    }

}
```

执行结果：

```
=====resource path=====
/Users/yu000hong/develop/hadoop/target/classes/com/yu000hong/test/
=====code source=====
/Users/yu000hong/develop/hadoop/target/classes/

=====resource path=====
file:/Users/yu000hong/.m2/repository/org/apache/hadoop/hadoop-mapreduce-client-core/3.2.1/hadoop-mapreduce-client-core-3.2.1.jar!/org/apache/hadoop/mapreduce/
=====code source=====
/Users/yu000hong/.m2/repository/org/apache/hadoop/hadoop-mapreduce-client-core/3.2.1/hadoop-mapreduce-client-core-3.2.1.jar

=====resource path=====
Exception in thread "main" java.lang.NullPointerException
	at com.yu000hong.test.Main.printResourcePath(Main.java:22)
	at com.yu000hong.test.Main.main(Main.java:16)
=====code source=====
Exception in thread "main" java.lang.NullPointerException
	at com.yu000hong.test.Main.printCodeSource(Main.java:28)
	at com.yu000hong.test.Main.main(Main.java:17)
```

从执行结果可以看出，两种方式还是有些区别。

**Main.class**

这个类是我们的执行类，是没有打包的，返回的是一个目录。对于`getResource()`方式来说，这个目录是当前类所在源文件的目录；对于`getCodeSource()`方式来说，这个目录是存放所有class文件的根目录。

**Mapper.class**

这个类是Hadoop类库中的类，是存在于Jar文件中的。对于`getResource()`方式来说，返回了Jar文件所在路径，同时附上了 **!/org/apache/hadoop/mapreduce/** 参数，也即Jar文件内部的路径位置；对于`getCodeSource()`方式来说，返回的就是这个Jar文件的路径位置。

**String.class**

这个是系统类，无论是使用哪种方式最后都会抛出异常，可能因为系统类受保护的原因(TODO)。

最后，我们来看看`Spring Boot Loader`是怎么使用的，[org.springframework.boot.loader.Launcher](https://github.com/spring-projects/spring-boot/blob/89555a874570e2215f69b180c8de42c91afd2b78/spring-boot-project/spring-boot-tools/spring-boot-loader/src/main/java/org/springframework/boot/loader/Launcher.java#L150)：

```java
protected final Archive createArchive() throws Exception {
    ProtectionDomain protectionDomain = getClass().getProtectionDomain();
    CodeSource codeSource = protectionDomain.getCodeSource();
    URI location = (codeSource != null) ? codeSource.getLocation().toURI() : null;
    String path = (location != null) ? location.getSchemeSpecificPart() : null;
    if (path == null) {
        throw new IllegalStateException("Unable to determine code source archive");
    }
    File root = new File(path);
    if (!root.exists()) {
        throw new IllegalStateException("Unable to determine code source archive from " + root);
    }
    return (root.isDirectory() ? new ExplodedArchive(root) : new JarFileArchive(root));
}
```