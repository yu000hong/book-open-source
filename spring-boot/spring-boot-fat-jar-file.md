# Spring Boot Fat Jar

### Jar file layout

A JAR (Java Archive) is a package file format typically used to aggregate many Java class files and associated metadata and resources (text, images, etc.) into one file to distribute application software or libraries on the Java platform.

They are built on the ZIP format and typically have a .jar file extension. We can imagine a .jar files as a zipped file(.zip) that is created by using WinZip software. Even, WinZip software can be used to used to extract the contents of a .jar . So you can use them for tasks such as lossless data compression, archiving, decompression, and archive unpacking.

There can be only one manifest file in an archive , and it always has the pathname: `META-INF/MANIFEST.MF`. The manifest file contains special meta information about files within the jar file.

**Executalbe Jar File**

只需要在`META-INF/MANIFEST.MF`文件中设置如下`Main-Class`属性即可。

```
Manifest-Version: 1.0
Main-Class: com.yu000hong.jar.Main
```

### Spring Boot Fat Jar

通过`jar -tf`命令查看Jar文件的结构：

```
├── BOOT-INF
│   ├── classes
│   │   ├── application-local.yml
│   │   ├── application-prod.yml
│   │   ├── application-test.yml
│   │   ├── application.yml
│   │   └── com
│   │       └── yu000hong
│   │           └── spring
│   │               ├── Application.class
│   │               ├── client
│   │               │   ├── impl
│   │               │   │   └── TopicClientImpl.class
│   │               │   └── ITopicClient.class
│   │               ├── domain
│   │               │   └── Topic.class
│   │               └── mapper
│   │                   ├── TopicMapper.class
│   │                   └── TopicMapper.xml
│   └── lib
│       ├── aggs-matrix-stats-client-5.6.3.jar
│       ├── animal-sniffer-annotations-1.18.jar
│       ├── antlr4-runtime-4.7.1.jar
│       ├── api-commons-4.264.jar
│       ├── api-data-4.906.jar
│       └── asm-7.3.jar
├── META-INF
│   ├── MANIFEST.MF
│   ├── maven
│   │   └── com.yu000hong.spring
│   │       └── fatjartest
│   │           ├── pom.properties
│   │           └── pom.xml
│   ├── org
│   │   └── apache
│   │       └── logging
│   │           └── log4j
│   │               └── core
│   │                   └── config
│   │                       └── plugins
│   │                           └── Log4j2Plugins.dat
│   └── spring-configuration-metadata.json
└── org
    └── springframework
        └── boot
            └── loader
                ├── archive
                │   ├── Archive.class
                │   ├── Archive$Entry.class
                │   ├── ExplodedArchive.class
                │   ├── JarFileArchive.class
                ├── data
                │   ├── RandomAccessData.class
                │   ├── RandomAccessDataFile$1.class
                │   ├── RandomAccessDataFile.class
                │   ├── RandomAccessDataFile$DataInputStream.class
                │   └── RandomAccessDataFile$FileAccess.class
                ├── ExecutableArchiveLauncher.class
                ├── jar
                │   ├── AsciiBytes.class
                │   ├── Bytes.class
                │   ├── JarURLConnection$1.class
                │   ├── JarURLConnection.class
                │   ├── StringSequence.class
                │   └── ZipInflaterInputStream.class
                ├── JarLauncher.class
                ├── LaunchedURLClassLoader.class
                ├── LaunchedURLClassLoader$UseFastConnectionExceptionsEnumeration.class
                ├── Launcher.class
                ├── MainMethodRunner.class
                ├── PropertiesLauncher$1.class
                ├── PropertiesLauncher$ArchiveEntryFilter.class
                ├── PropertiesLauncher.class
                ├── PropertiesLauncher$PrefixMatchingArchiveFilter.class
                ├── util
                │   └── SystemPropertyUtils.class
                └── WarLauncher.class
```

我们再看看`META-INF/MANIFEST.MF`文件的内容：

```
Manifest-Version: 1.0
Start-Class: com.yu000hong.spring.Application
Spring-Boot-Classes: BOOT-INF/classes/
Spring-Boot-Lib: BOOT-INF/lib/
Build-Jdk-Spec: 1.8
Spring-Boot-Version: 2.1.9.RELEASE
Created-By: Maven Archiver 3.4.0
Main-Class: org.springframework.boot.loader.JarLauncher
```

从目录结构和Manifest文件可以看出，**Spring Boot Fat Jar**的可执行入口为：`org.springframework.boot.loader.JarLauncher`。同时，多了`BOOT-INF/classes`和`BOOT-INF/lib`两个目录，这两个目录分别对应了我们自己的代码和依赖库Jar文件。如何加载`BOOT-INF/classes`目录下的class文件及其它资源，如何加载`BOOT-INF/lib`目录下嵌套的Jar文件，这些都是`JarLauncher`的作用了。

我们看Manifest文件里多了几个属性：

- Spring-Boot-Classes
- Spring-Boot-Lib
- Start-Class
- Build-Jdk-Spec
- Spring-Boot-Version

TODO


