# Lettuce报错：HashedWheelTimer.release() was not called before it's garbage-collected

启动Spring Boot应用之后，发现报错如下：

```
2021-05-18 08:46:37.654	ERROR	io.netty.util.ResourceLeakDetector	io.netty.util.ResourceLeakDetector:317	LEAK: HashedWheelTimer.release() was not called before it's garbage-collecte
d. See https://netty.io/wiki/reference-counted-objects.html for more information.
Recent access records:
Created at:
	io.netty.util.HashedWheelTimer.<init>(HashedWheelTimer.java:284)
	io.netty.util.HashedWheelTimer.<init>(HashedWheelTimer.java:217)
	io.netty.util.HashedWheelTimer.<init>(HashedWheelTimer.java:196)
	io.netty.util.HashedWheelTimer.<init>(HashedWheelTimer.java:178)
	io.netty.util.HashedWheelTimer.<init>(HashedWheelTimer.java:162)
	io.lettuce.core.resource.DefaultClientResources.<init>(DefaultClientResources.java:169)
	io.lettuce.core.resource.DefaultClientResources$Builder.build(DefaultClientResources.java:532)
	io.lettuce.core.resource.DefaultClientResources.create(DefaultClientResources.java:233)
	com.weibo.oasis.common.configuration.redis.RedisPostProcessor$CustomStringRedisTemplate.<init>(RedisPostProcessor.java:123)
	sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
	sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62)
	sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
	java.lang.reflect.Constructor.newInstance(Constructor.java:423)
	org.springframework.beans.BeanUtils.instantiateClass(BeanUtils.java:172)
	org.springframework.beans.factory.support.SimpleInstantiationStrategy.instantiate(SimpleInstantiationStrategy.java:117)
	org.springframework.beans.factory.support.ConstructorResolver.instantiate(ConstructorResolver.java:300)
	org.springframework.beans.factory.support.ConstructorResolver.autowireConstructor(ConstructorResolver.java:285)
	org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.autowireConstructor(AbstractAutowireCapableBeanFactory.java:1341)
	org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBeanInstance(AbstractAutowireCapableBeanFactory.java:1187)
	org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:555)
	org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(AbstractAutowireCapableBeanFactory.java:515)
	org.springframework.beans.factory.support.AbstractBeanFactory.lambda$doGetBean$0(AbstractBeanFactory.java:320)
	org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:222)
	org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:318)
	org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:204)
	org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.resolveBeanByName(AbstractAutowireCapableBeanFactory.java:452)
	org.springframework.context.annotation.CommonAnnotationBeanPostProcessor.autowireResource(CommonAnnotationBeanPostProcessor.java:527)
	org.springframework.context.annotation.CommonAnnotationBeanPostProcessor.getResource(CommonAnnotationBeanPostProcessor.java:497)
	org.springframework.context.annotation.CommonAnnotationBeanPostProcessor$ResourceElement.getResourceToInject(CommonAnnotationBeanPostProcessor.java:637)
	org.springframework.beans.factory.annotation.InjectionMetadata$InjectedElement.inject(InjectionMetadata.java:180)
	org.springframework.beans.factory.annotation.InjectionMetadata.inject(InjectionMetadata.java:90)
	org.springframework.context.annotation.CommonAnnotationBeanPostProcessor.postProcessProperties(CommonAnnotationBeanPostProcessor.java:322)
	org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.populateBean(AbstractAutowireCapableBeanFactory.java:1411)
	org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:592)
	org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(AbstractAutowireCapableBeanFactory.java:515)
	org.springframework.beans.factory.support.AbstractBeanFactory.lambda$doGetBean$0(AbstractBeanFactory.java:320)
	org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:222)
	org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:318)
	org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:199)
	org.springframework.beans.factory.support.ConstructorResolver.instantiateUsingFactoryMethod(ConstructorResolver.java:392)
	org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.instantiateUsingFactoryMethod(AbstractAutowireCapableBeanFactory.java:1321)
	org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBeanInstance(AbstractAutowireCapableBeanFactory.java:1160)
	org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:555)
	org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(AbstractAutowireCapableBeanFactory.java:515)
	org.springframework.beans.factory.support.AbstractBeanFactory.lambda$doGetBean$0(AbstractBeanFactory.java:320)
	org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:222)
	org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:318)
	org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:204)
	org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.resolveBeanByName(AbstractAutowireCapableBeanFactory.java:452)
	org.springframework.context.annotation.CommonAnnotationBeanPostProcessor.autowireResource(CommonAnnotationBeanPostProcessor.java:527)
	org.springframework.context.annotation.CommonAnnotationBeanPostProcessor.getResource(CommonAnnotationBeanPostProcessor.java:497)
	org.springframework.context.annotation.CommonAnnotationBeanPostProcessor$ResourceElement.getResourceToInject(CommonAnnotationBeanPostProcessor.java:637)
	org.springframework.beans.factory.annotation.InjectionMetadata$InjectedElement.inject(InjectionMetadata.java:180)
	org.springframework.beans.factory.annotation.InjectionMetadata.inject(InjectionMetadata.java:90)
	org.springframework.context.annotation.CommonAnnotationBeanPostProcessor.postProcessProperties(CommonAnnotationBeanPostProcessor.java:322)
	org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.populateBean(AbstractAutowireCapableBeanFactory.java:1411)
	org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:592)
	org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(AbstractAutowireCapableBeanFactory.java:515)
	org.springframework.beans.factory.support.AbstractBeanFactory.lambda$doGetBean$0(AbstractBeanFactory.java:320)
	org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:222)
	org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:318)
	org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:199)
	org.springframework.beans.factory.config.DependencyDescriptor.resolveCandidate(DependencyDescriptor.java:277)
	org.springframework.beans.factory.support.DefaultListableBeanFactory.addCandidateEntry(DefaultListableBeanFactory.java:1478)
	org.springframework.beans.factory.support.DefaultListableBeanFactory.findAutowireCandidates(DefaultListableBeanFactory.java:1435)
	org.springframework.beans.factory.support.DefaultListableBeanFactory.resolveMultipleBeans(DefaultListableBeanFactory.java:1281)
	org.springframework.beans.factory.support.DefaultListableBeanFactory.doResolveDependency(DefaultListableBeanFactory.java:1213)
	org.springframework.beans.factory.support.DefaultListableBeanFactory$DependencyObjectProvider.resolveStream(DefaultListableBeanFactory.java:1936)
	org.springframework.beans.factory.support.DefaultListableBeanFactory$DependencyObjectProvider.orderedStream(DefaultListableBeanFactory.java:1930)
	org.springframework.boot.actuate.autoconfigure.metrics.MeterRegistryConfigurer.addBinders(MeterRegistryConfigurer.java:84)
	org.springframework.boot.actuate.autoconfigure.metrics.MeterRegistryConfigurer.configure(MeterRegistryConfigurer.java:66)
	org.springframework.boot.actuate.autoconfigure.metrics.MeterRegistryPostProcessor.postProcessAfterInitialization(MeterRegistryPostProcessor.java:64)
	org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.applyBeanPostProcessorsAfterInitialization(AbstractAutowireCapableBeanFactory.java:429)
	org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.initializeBean(AbstractAutowireCapableBeanFactory.java:1782)
	org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:593)
	org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(AbstractAutowireCapableBeanFactory.java:515)
	org.springframework.beans.factory.support.AbstractBeanFactory.lambda$doGetBean$0(AbstractBeanFactory.java:320)
	org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:222)
	org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:318)
	org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:199)
	org.springframework.beans.factory.config.DependencyDescriptor.resolveCandidate(DependencyDescriptor.java:277)
	org.springframework.beans.factory.support.DefaultListableBeanFactory.doResolveDependency(DefaultListableBeanFactory.java:1255)
	org.springframework.beans.factory.support.DefaultListableBeanFactory.resolveDependency(DefaultListableBeanFactory.java:1175)
	org.springframework.beans.factory.support.ConstructorResolver.resolveAutowiredArgument(ConstructorResolver.java:857)
	org.springframework.beans.factory.support.ConstructorResolver.createArgumentArray(ConstructorResolver.java:760)
	org.springframework.beans.factory.support.ConstructorResolver.instantiateUsingFactoryMethod(ConstructorResolver.java:509)
	org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.instantiateUsingFactoryMethod(AbstractAutowireCapableBeanFactory.java:1321)
	org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBeanInstance(AbstractAutowireCapableBeanFactory.java:1160)
	org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:555)
	org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(AbstractAutowireCapableBeanFactory.java:515)
	org.springframework.beans.factory.support.AbstractBeanFactory.lambda$doGetBean$0(AbstractBeanFactory.java:320)
	org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:222)
	org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:318)
	org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:204)
	org.springframework.boot.web.servlet.ServletContextInitializerBeans.getOrderedBeansOfType(ServletContextInitializerBeans.java:211)
	org.springframework.boot.web.servlet.ServletContextInitializerBeans.getOrderedBeansOfType(ServletContextInitializerBeans.java:202)
	org.springframework.boot.web.servlet.ServletContextInitializerBeans.addServletContextInitializerBeans(ServletContextInitializerBeans.java:96)
	org.springframework.boot.web.servlet.ServletContextInitializerBeans.<init>(ServletContextInitializerBeans.java:85)
	org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext.getServletContextInitializerBeans(ServletWebServerApplicationContext.java:253)
	org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext.selfInitialize(ServletWebServerApplicationContext.java:227)
	org.springframework.boot.web.embedded.tomcat.TomcatStarter.onStartup(TomcatStarter.java:53)
	org.apache.catalina.core.StandardContext.startInternal(StandardContext.java:5135)
	org.apache.catalina.util.LifecycleBase.start(LifecycleBase.java:183)
	org.apache.catalina.core.ContainerBase$StartChild.call(ContainerBase.java:1384)
	org.apache.catalina.core.ContainerBase$StartChild.call(ContainerBase.java:1374)
	java.util.concurrent.FutureTask.run(FutureTask.java:266)
	org.apache.tomcat.util.threads.InlineExecutorService.execute(InlineExecutorService.java:75)
	java.util.concurrent.AbstractExecutorService.submit(AbstractExecutorService.java:134)
	org.apache.catalina.core.ContainerBase.startInternal(ContainerBase.java:909)
	org.apache.catalina.core.StandardHost.startInternal(StandardHost.java:841)
	org.apache.catalina.util.LifecycleBase.start(LifecycleBase.java:183)
	org.apache.catalina.core.ContainerBase$StartChild.call(ContainerBase.java:1384)
	org.apache.catalina.core.ContainerBase$StartChild.call(ContainerBase.java:1374)
	java.util.concurrent.FutureTask.run(FutureTask.java:266)
	org.apache.tomcat.util.threads.InlineExecutorService.execute(InlineExecutorService.java:75)
	java.util.concurrent.AbstractExecutorService.submit(AbstractExecutorService.java:134)
	org.apache.catalina.core.ContainerBase.startInternal(ContainerBase.java:909)
	org.apache.catalina.core.StandardEngine.startInternal(StandardEngine.java:262)
	org.apache.catalina.util.LifecycleBase.start(LifecycleBase.java:183)
	org.apache.catalina.core.StandardService.startInternal(StandardService.java:421)
	org.apache.catalina.util.LifecycleBase.start(LifecycleBase.java:183)
	org.apache.catalina.core.StandardServer.startInternal(StandardServer.java:932)
	org.apache.catalina.util.LifecycleBase.start(LifecycleBase.java:183)
	org.apache.catalina.startup.Tomcat.start(Tomcat.java:459)
	org.springframework.boot.web.embedded.tomcat.TomcatWebServer.initialize(TomcatWebServer.java:105)
	org.springframework.boot.web.embedded.tomcat.TomcatWebServer.<init>(TomcatWebServer.java:86)
	org.springframework.boot.web.embedded.tomcat.TomcatServletWebServerFactory.getTomcatWebServer(TomcatServletWebServerFactory.java:416)
	org.springframework.boot.web.embedded.tomcat.TomcatServletWebServerFactory.getWebServer(TomcatServletWebServerFactory.java:180)
	org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext.createWebServer(ServletWebServerApplicationContext.java:180)
	org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext.onRefresh(ServletWebServerApplicationContext.java:153)
	org.springframework.context.support.AbstractApplicationContext.refresh(AbstractApplicationContext.java:543)
	org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext.refresh(ServletWebServerApplicationContext.java:141)
	org.springframework.boot.SpringApplication.refresh(SpringApplication.java:744)
	org.springframework.boot.SpringApplication.refreshContext(SpringApplication.java:391)
	org.springframework.boot.SpringApplication.run(SpringApplication.java:312)
	org.springframework.boot.SpringApplication.run(SpringApplication.java:1215)
	org.springframework.boot.SpringApplication.run(SpringApplication.java:1204)
	com.weibo.oasis.task.Application.main(Application.java:31)
	sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	java.lang.reflect.Method.invoke(Method.java:498)
	org.springframework.boot.loader.MainMethodRunner.run(MainMethodRunner.java:48)
	org.springframework.boot.loader.Launcher.launch(Launcher.java:87)
	org.springframework.boot.loader.Launcher.launch(Launcher.java:51)
	org.springframework.boot.loader.JarLauncher.main(JarLauncher.java:52)   
```

报错日志里面，让我们参考如下页面：[https://netty.io/wiki/reference-counted-objects.html](https://netty.io/wiki/reference-counted-objects.html)。

网上搜索结果：

> If you use many RedisClient instances in your application - you need to share between them ClientResources object.
>
>The typical way to create RedisClient is:
>
> ```java
> RedisClient.create("someUrl");
> ```
>
> But this way of object creation is not recommended when you want create multiple instances. Inside RedisClient there is field with heavy object: ClientResources When you are creating many instances of RedisClient then error can be produced during runtime
>
> ```python
> LEAK: You are creating too many HashedWheelTimer instances.  HashedWheelTimer is a shared resource that must be reused across the JVM,so that only a few instances are created.
> ```
>
> What you can do is to create one shared ClientResources in your application. And pass this object to the constructor RedisClient.
>
>For example:
>
> ```java
>   ClientResources sharedResources = DefaultClientResources.create();
>    //and use this between many RedisClients i.e.:
>
>   RedisClient first = RedisClient.create(sharedResources, "someUrl");
>   RedisClient second = RedisClient.create(sharedResources, "someUrl_222");
>   RedisClient third = RedisClient.create(sharedResources, "someUrl_3333");
> ```
>
> For more deatils, please visit [Configuring Client resources](https://github.com/lettuce-io/lettuce-core/wiki/Configuring-Client-resources).
>

