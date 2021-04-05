# ContextLoader

> Strategy interface for loading an application context for an integration test managed by the Spring TestContext Framework.
> Clients of a ContextLoader should call processLocations() prior to calling loadContext() in case the ContextLoader provides custom support for modifying or generating locations. The results of processLocations() should then be supplied to loadContext().

```
ContextLoader
    |---SmartContextLoader
         |---AbstractDelegatingSmartContextLoader
         |      |---DelegatingSmartContextLoader
         |      |---WebDelegatingSmartContextLoader
         |---AbstractContextLoader
                |---AbstractGenericWebContextLoader
                |      |---AnnotationConfigWebContextLoader
                |      |---GenericXmlWebContextLoader
                |              |---GenericGroovyXmlWebContextLoader
                |---AbstractGenericContextLoader
                       |---AnnotationConfigContextLoader
                       |---GenericXmlContextLoader
                               |---GenericGroovyXmlContextLoader
```

### AbstractContextLoader

##### processLocations

```java
public final String[] processLocations(Class<?> clazz, String... locations) {
    return (ObjectUtils.isEmpty(locations) && isGenerateDefaultLocations()) ?
            generateDefaultLocations(clazz) : modifyLocations(clazz, locations);
}
```

如果提供了**locations**参数，那么根据其使用的是绝对地址还是相对地址会区别处理：
- 绝对地址：在location前面加**classpath:**，即`classpath:$location$`
- 相对地址：加**classpath:**，同时还要处理clazz的路径，即`classpath:/package/of/clazz/Clazz/$location$`

如果没有提供**locations**参数，那么使用默认值，即：`classpath:/your/package/directory/YourClass$SUFFIX$`。

例如：

```java
package org.test;

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

并且使用XML方式，那么其locations默认值为：`classpath:/org/test/DefaultTest-context.xml`。

##### processContextConfiguration

```java
public void processContextConfiguration(ContextConfigurationAttributes configAttributes) {
    String[] processedLocations =
            processLocations(configAttributes.getDeclaringClass(), configAttributes.getLocations());
    configAttributes.setLocations(processedLocations);
}
```

处理locations参数，并将其重新设置到configAttributes变量里面去。

##### invokeApplicationContextInitializers

从MergedContextConfiguration中获取我们设置的ApplicationContextInitializer，然后实例化后调用其initialize()方法进行初始化。

##### prepareContext

```java
protected void prepareContext(ConfigurableApplicationContext context, MergedContextConfiguration mergedConfig) {
    context.getEnvironment().setActiveProfiles(mergedConfig.getActiveProfiles());
    TestPropertySourceUtils.addPropertiesFilesToEnvironment(context, mergedConfig.getPropertySourceLocations());
    TestPropertySourceUtils.addInlinedPropertiesToEnvironment(context, mergedConfig.getPropertySourceProperties());
    invokeApplicationContextInitializers(context, mergedConfig);
}
```

准备ApplicationContext：
- 设置activeProfiles
- 添加`@IfPropertySource`中locations指定的属性文件到environment
- 添加`@IfPropertySource`中properties指定的属性到environment
- 调用所有设置的ApplicationContextInitializer

### AbstractGenericContextLoader

##### customizeContext

```java
protected void customizeContext(GenericApplicationContext context) {
}
```

##### customizeBeanFactory

```java
protected void customizeBeanFactory(DefaultListableBeanFactory beanFactory) {
}
```

##### createBeanDefinitionReader

```java
abstract BeanDefinitionReader createBeanDefinitionReader(GenericApplicationContext context);
```

##### loadBeanDefinitions

```java
protected void loadBeanDefinitions(GenericApplicationContext context, MergedContextConfiguration mergedConfig) {
    createBeanDefinitionReader(context).loadBeanDefinitions(mergedConfig.getLocations());
}
```

##### loadContext

```java
public final ConfigurableApplicationContext loadContext(String... locations) throws Exception {
    GenericApplicationContext context = new GenericApplicationContext();
    prepareContext(context);
    customizeBeanFactory(context.getDefaultListableBeanFactory());
    createBeanDefinitionReader(context).loadBeanDefinitions(locations);
    AnnotationConfigUtils.registerAnnotationConfigProcessors(context);
    customizeContext(context);
    context.refresh();
    context.registerShutdownHook();
    return context;
}
```

```java
public final ConfigurableApplicationContext loadContext(MergedContextConfiguration mergedConfig) throws Exception {
    validateMergedContextConfiguration(mergedConfig);
    GenericApplicationContext context = new GenericApplicationContext();
    ApplicationContext parent = mergedConfig.getParentApplicationContext();
    if (parent != null) {
        context.setParent(parent);
    }
    prepareContext(context);
    prepareContext(context, mergedConfig);
    customizeBeanFactory(context.getDefaultListableBeanFactory());
    loadBeanDefinitions(context, mergedConfig);
    AnnotationConfigUtils.registerAnnotationConfigProcessors(context);
    customizeContext(context);
    customizeContext(context, mergedConfig);
    context.refresh();
    context.registerShutdownHook();
    return context;
}
```

### AnnotationConfigContextLoader

##### processContextConfiguration

```java
public void processContextConfiguration(ContextConfigurationAttributes configAttributes) {
    if (!configAttributes.hasClasses() && isGenerateDefaultLocations()) {
        configAttributes.setClasses(detectDefaultConfigurationClasses(configAttributes.getDeclaringClass()));
    }
}
```

如果`@ContextConfiguration`没有设置classes属性，那么就会在测试类的内部里面去找满足如下条件的类：
- 静态内部类
- 非final类
- 非private类
- 带有`@Configuration`注解

##### locations

因为`AnnotationConfigContextLoader`用于根据注解的方式加载，所以不支持locations属性：

```java
protected void validateMergedContextConfiguration(MergedContextConfiguration mergedConfig) {
    //如果配置了locations属性，会抛出IllegalStateException异常
    if (mergedConfig.hasLocations()) {
        String msg = String.format("Test class [%s] has been configured with @ContextConfiguration's 'locations' " +
                        "(or 'value') attribute %s, but %s does not support resource locations.",
                mergedConfig.getTestClass().getName(), ObjectUtils.nullSafeToString(mergedConfig.getLocations()),
                getClass().getSimpleName());
        logger.error(msg);
        throw new IllegalStateException(msg);
    }
}
protected String[] modifyLocations(Class<?> clazz, String... locations) {
    throw new UnsupportedOperationException(
            "AnnotationConfigContextLoader does not support the modifyLocations(Class, String...) method");
}
protected String[] generateDefaultLocations(Class<?> clazz) {
    throw new UnsupportedOperationException(
            "AnnotationConfigContextLoader does not support the generateDefaultLocations(Class) method");
}
protected String getResourceSuffix() {
    throw new UnsupportedOperationException(
            "AnnotationConfigContextLoader does not support the getResourceSuffix() method");
}
```

##### loadBeanDefinitions

```java
protected void loadBeanDefinitions(GenericApplicationContext context, MergedContextConfiguration mergedConfig) {
    Class<?>[] componentClasses = mergedConfig.getClasses();
    if (logger.isDebugEnabled()) {
        logger.debug("Registering component classes: " + ObjectUtils.nullSafeToString(componentClasses));
    }
    new AnnotatedBeanDefinitionReader(context).register(componentClasses);
}
```

### GenericXmlContextLoader

> Concrete implementation of `AbstractGenericContextLoader` that reads bean definitions from XML resources.
> Default resource locations are detected using the suffix **-context.xml**.

##### createBeanDefinitionReader

```java
protected BeanDefinitionReader createBeanDefinitionReader(GenericApplicationContext context) {
    return new XmlBeanDefinitionReader(context);
}
```

##### getResourceSuffix

```java
protected String getResourceSuffix() {
    return "-context.xml";
}
```

##### validateMergedContextConfiguration

```java
protected void validateMergedContextConfiguration(MergedContextConfiguration mergedConfig) {
    //如果配置了classes属性，那么会抛出IllegalStateException异常
    if (mergedConfig.hasClasses()) {
        String msg = String.format(
            "Test class [%s] has been configured with @ContextConfiguration's 'classes' attribute %s, "
                    + "but %s does not support annotated classes.", mergedConfig.getTestClass().getName(),
            ObjectUtils.nullSafeToString(mergedConfig.getClasses()), getClass().getSimpleName());
        logger.error(msg);
        throw new IllegalStateException(msg);
    }
}
```

### GenericGroovyXmlContextLoader

> Concrete implementation of `AbstractGenericContextLoader` that reads bean definitions from Groovy scripts and XML configuration files.
> Default resource locations are detected using the suffixes "-context.xml" and "Context.groovy".

##### loadBeanDefinitions

```java
protected void loadBeanDefinitions(GenericApplicationContext context, MergedContextConfiguration mergedConfig) {
    new GroovyBeanDefinitionReader(context).loadBeanDefinitions(mergedConfig.getLocations());
}
```

##### getResourceSuffixes

```java
protected String[] getResourceSuffixes() {
    return new String[] { super.getResourceSuffix(), "Context.groovy" };
}
```

### AbstractGenericWebContextLoader

##### loadContext

```java
public final ConfigurableApplicationContext loadContext(MergedContextConfiguration mergedConfig) throws Exception {
    WebMergedContextConfiguration webMergedConfig = (WebMergedContextConfiguration) mergedConfig;
    validateMergedContextConfiguration(webMergedConfig);
    GenericWebApplicationContext context = new GenericWebApplicationContext();
    ApplicationContext parent = mergedConfig.getParentApplicationContext();
    if (parent != null) {
        context.setParent(parent);
    }
    configureWebResources(context, webMergedConfig);
    prepareContext(context, webMergedConfig);
    customizeBeanFactory(context.getDefaultListableBeanFactory(), webMergedConfig);
    loadBeanDefinitions(context, webMergedConfig);
    AnnotationConfigUtils.registerAnnotationConfigProcessors(context);
    customizeContext(context, webMergedConfig);
    context.refresh();
    context.registerShutdownHook();
    return context;
}
```

`@WebAppConfiguration`的loadContext()跟非WEB环境的唯一区别在：`configureWebResources()`。

##### configureWebResources

```java
protected void configureWebResources(GenericWebApplicationContext context,
        WebMergedContextConfiguration webMergedConfig) {
    ApplicationContext parent = context.getParent();
    // If the WebApplicationContext has no parent or the parent is not a WebApplicationContext,
    // set the current context as the root WebApplicationContext:
    if (parent == null || (!(parent instanceof WebApplicationContext))) {
        String resourceBasePath = webMergedConfig.getResourceBasePath();
        ResourceLoader resourceLoader = (resourceBasePath.startsWith(ResourceLoader.CLASSPATH_URL_PREFIX) ?
                new DefaultResourceLoader() : new FileSystemResourceLoader());
        ServletContext servletContext = new MockServletContext(resourceBasePath, resourceLoader);
        servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, context);
        context.setServletContext(servletContext);
    }
    else {
        ServletContext servletContext = null;
        // Find the root WebApplicationContext
        while (parent != null) {
            if (parent instanceof WebApplicationContext && !(parent.getParent() instanceof WebApplicationContext)) {
                servletContext = ((WebApplicationContext) parent).getServletContext();
                break;
            }
            parent = parent.getParent();
        }
        Assert.state(servletContext != null, "Failed to find root WebApplicationContext in the context hierarchy");
        context.setServletContext(servletContext);
    }
}
```

可以看出，主要目的是要给WEB环境设置一个`MockServletContext`。

### AnnotationConfigWebContextLoader

与`AnnotationConfigContextLoader`功能基本一致，除了`AnnotationConfigWebContextLoader`继承自`AbstractGenericWebContextLoader`，继承了WEB环境下的**MockServletContext**。

### GenericXmlWebContextLoader

与`GenericXmlContextLoader`功能基本一致，除了`GenericXmlWebContextLoader`继承自`AbstractGenericWebContextLoader`，继承了WEB环境下的**MockServletContext**。

### GenericGroovyXmlWebContextLoader

与`GenericGroovyXmlContextLoader`功能基本一致，除了`GenericGroovyXmlWebContextLoader`从祖父类`AbstractGenericWebContextLoader`继承了WEB环境下的**MockServletContext**。

### AbstractDelegatingSmartContextLoader

使用代理模式根据具体的环境选择适合的SmartContextLoader。

```java
public void processContextConfiguration(final ContextConfigurationAttributes configAttributes) {
    // If the original locations or classes were not empty, there's no
    // need to bother with default detection checks; just let the
    // appropriate loader process the configuration.
    if (configAttributes.hasLocations()) {
        delegateProcessing(getXmlLoader(), configAttributes);
    }
    else if (configAttributes.hasClasses()) {
        delegateProcessing(getAnnotationConfigLoader(), configAttributes);
    }
    else {
        // Else attempt to detect defaults...

        // Let the XML loader process the configuration.
        delegateProcessing(getXmlLoader(), configAttributes);
        boolean xmlLoaderDetectedDefaults = configAttributes.hasLocations();
        if (xmlLoaderDetectedDefaults) {
            if (logger.isInfoEnabled()) {
                logger.info(String.format("%s detected default locations for context configuration %s.",
                        name(getXmlLoader()), configAttributes));
            }
        }
        Assert.state(!configAttributes.hasClasses(), () -> String.format(
                "%s should NOT have detected default configuration classes for context configuration %s.",
                name(getXmlLoader()), configAttributes));

        // Now let the annotation config loader process the configuration.
        delegateProcessing(getAnnotationConfigLoader(), configAttributes);
        if (configAttributes.hasClasses()) {
            if (logger.isInfoEnabled()) {
                logger.info(String.format("%s detected default configuration classes for context configuration %s.",
                        name(getAnnotationConfigLoader()), configAttributes));
            }
        }
        Assert.state(xmlLoaderDetectedDefaults || !configAttributes.hasLocations(), () -> String.format(
                "%s should NOT have detected default locations for context configuration %s.",
                name(getAnnotationConfigLoader()), configAttributes));

        if (configAttributes.hasLocations() && configAttributes.hasClasses()) {
            String msg = String.format(
                    "Configuration error: both default locations AND default configuration classes " +
                    "were detected for context configuration %s; configure one or the other, but not both.",
                    configAttributes);
            logger.error(msg);
            throw new IllegalStateException(msg);
        }
    }
}
```

```java
public ApplicationContext loadContext(MergedContextConfiguration mergedConfig) throws Exception {
    SmartContextLoader[] candidates = {getXmlLoader(), getAnnotationConfigLoader()};
    for (SmartContextLoader loader : candidates) {
        // Determine if each loader can load a context from the mergedConfig. If it
        // can, let it; otherwise, keep iterating.
        if (supports(loader, mergedConfig)) {
            return delegateLoading(loader, mergedConfig);
        }
    }
    // If neither of the candidates supports the mergedConfig based on resources but
    // ACIs or customizers were declared, then delegate to the annotation config
    // loader.
    if (!mergedConfig.getContextInitializerClasses().isEmpty() || !mergedConfig.getContextCustomizers().isEmpty()) {
        return delegateLoading(getAnnotationConfigLoader(), mergedConfig);
    }

    // else...
    throw new IllegalStateException(String.format(
            "Neither %s nor %s was able to load an ApplicationContext from %s.",
            name(getXmlLoader()), name(getAnnotationConfigLoader()), mergedConfig));
}
```

##### DelegatingSmartContextLoader

```java
public DelegatingSmartContextLoader() {
    if (groovyPresent) {
        try {
            Class<?> loaderClass = ClassUtils.forName(GROOVY_XML_CONTEXT_LOADER_CLASS_NAME,
                DelegatingSmartContextLoader.class.getClassLoader());
            this.xmlLoader = (SmartContextLoader) BeanUtils.instantiateClass(loaderClass);
        }
        catch (Throwable ex) {
            throw new IllegalStateException("Failed to enable support for Groovy scripts; "
                    + "could not load class: " + GROOVY_XML_CONTEXT_LOADER_CLASS_NAME, ex);
        }
    }
    else {
        this.xmlLoader = new GenericXmlContextLoader();
    }
    this.annotationConfigLoader = new AnnotationConfigContextLoader();
}
```

##### WebDelegatingSmartContextLoader

```java
public WebDelegatingSmartContextLoader() {
    if (groovyPresent) {
        try {
            Class<?> loaderClass = ClassUtils.forName(GROOVY_XML_WEB_CONTEXT_LOADER_CLASS_NAME,
                WebDelegatingSmartContextLoader.class.getClassLoader());
            this.xmlLoader = (SmartContextLoader) BeanUtils.instantiateClass(loaderClass);
        }
        catch (Throwable ex) {
            throw new IllegalStateException("Failed to enable support for Groovy scripts; "
                    + "could not load class: " + GROOVY_XML_WEB_CONTEXT_LOADER_CLASS_NAME, ex);
        }
    }
    else {
        this.xmlLoader = new GenericXmlWebContextLoader();
    }
    this.annotationConfigLoader = new AnnotationConfigWebContextLoader();
}
```

