# MyBatis

### 使用流程

#### 从XML文件配置

```java
String resource = "org/mybatis/example/mybatis-config.xml";
InputStream inputStream = Resources.getResourceAsStream(resource);
SqlSessionFactory sqlSessionFactory =
  new SqlSessionFactoryBuilder().build(inputStream);
```

#### Java代码配置

```java
DataSource dataSource = BlogDataSourceFactory.getBlogDataSource();
TransactionFactory transactionFactory =
  new JdbcTransactionFactory();
Environment environment =
  new Environment("development", transactionFactory, dataSource);
Configuration configuration = new Configuration(environment);
configuration.addMapper(BlogMapper.class);
SqlSessionFactory sqlSessionFactory =
  new SqlSessionFactoryBuilder().build(configuration);
```

#### 注解配置

TODO

### SqlSessionFactory

SqlSessionFactory这是比较重要的一个类，有了它我们才能获取SqlSession，后续所有的操作都是基于这个SqlSession的，比如：

- 获取Mapper类：session.getMapper(Class type)
- 直接执行statement：session.selectOne(String statement)

下面我们就着重看看如何从XML文件构建SqlSessionFactory的。

主要的类就是`XMLConfigBuilder`，由它的`parse()`方法解析XML文件最后生成返回`Configuration`对象。

看看主要代码：

```java

private void parseConfiguration(XNode root) {
    try {
        // issue #117 read properties first
        propertiesElement(root.evalNode("properties"));
        Properties settings = settingsAsProperties(root.evalNode("settings"));
        loadCustomVfs(settings);
        loadCustomLogImpl(settings);
        typeAliasesElement(root.evalNode("typeAliases"));
        pluginElement(root.evalNode("plugins"));
        objectFactoryElement(root.evalNode("objectFactory"));
        objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
        reflectorFactoryElement(root.evalNode("reflectorFactory"));
        settingsElement(settings);
        // read it after objectFactory and objectWrapperFactory issue #631
        environmentsElement(root.evalNode("environments"));
        databaseIdProviderElement(root.evalNode("databaseIdProvider"));
        typeHandlerElement(root.evalNode("typeHandlers"));
        mapperElement(root.evalNode("mappers"));
    } catch (Exception e) {
        throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
    }
}
```

XML文件一共涉及如下几个子节点：

- properties
- settings
- typeAliases
- typeHandlers
- objectFactory
- plugin
- environments
- databaseIdProvider
- mappers

#### properties
