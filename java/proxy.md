# Java Proxy

### Proxy Design Pattern

Proxy is a structural design pattern that provides an object that acts as a substitute for a real service object used by a client. A proxy receives client requests, does some work (access control, caching, etc.) and then passes the request to a service object. We create and use proxy objects when we want to add or modify some functionality of an already existing class. The proxy object is used instead of the original one. Usually, the proxy objects have the same methods as the original one and in Java proxy classes usually extend the original class. The proxy has a handle to the original object and can call the method on that.

Dynamic proxies can be used for many different purposes:

- logging when a method starts and stops
- perform extra checks on arguments
- mocking the behavior of the original class
- implement lazy access to costly resources
- database connection and transaction management
- AOP-like method intercepting purposes

### java.lang.reflect.Proxy

**Creating Proxies**:

`Proxy.newProxyInstance()` method takes 3 parameters:

- The **ClassLoader** that is to "load" the dynamic proxy class.
- An array of **interfaces** to implement.
- An **InvocationHandler** to forward all methods calls on the proxy to.

**Example**:

```java
InvocationHandler handler = new MyInvocationHandler();
MyInterface proxy = (MyInterface) Proxy.newProxyInstance(
                            MyInterface.class.getClassLoader(),
                            new Class[] { MyInterface.class },
                            handler);
```

Proxy提供了4个公共方法，而且都是静态方法：

- boolean isProxyClass(Class<?> cl)
- InvocationHandler getInvocationHandler(Object proxy)
- Class<?> getProxyClass(ClassLoader loader, Class<?>... interfaces)
- Object newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h)

**isProxyClass**

判断某个Class是否是Java代理类，判断的两个条件：

- 必须是Proxy的子类
- 必须在Proxy的缓存里面

源码：

```java
public static boolean isProxyClass(Class<?> cl) {
  return Proxy.class.isAssignableFrom(cl) && proxyClassCache.containsValue(cl);
}
```

**getInvocationHandler**

获取代理类的内部InvocationHandler字段，源码：

```java
public static InvocationHandler getInvocationHandler(Object proxy)
        throws IllegalArgumentException {
  if (!isProxyClass(proxy.getClass())) {
    throw new IllegalArgumentException("not a proxy instance");
  }
  final Proxy p = (Proxy) proxy;
  final InvocationHandler ih = p.h;
  return ih;
}
```

**getProxyClass**

Proxy首先会从缓存中去查找是否已经生成了对应的代理类，缓存的Key就是ClassLoader和Interfaces，缓存的Value就是对应的Class。如果有的话直接返回，如果没有的话，会调用`ProxyGenerator.generateProxyClass(proxyName, interfaces, accessFlags)`去生成并使用参数提供的ClassLoader加载代理类。

**newProxyInstance**

首先使用`getProxyClass()`生成对应的代理类，然后使用反射实例化一个代理对象，源码：

```java
protected Proxy(InvocationHandler h) {
  Objects.requireNonNull(h);
  this.h = h;
}
```

```java
public static Object newProxyInstance(
        ClassLoader loader,
        Class<?>[] interfaces,
        InvocationHandler h) throws IllegalArgumentException {
  final Class<?>[] intfs = interfaces.clone();
  Class<?> cl = getProxyClass0(loader, intfs);
  final Constructor<?> cons = cl.getConstructor(new Class<?>[]{InvocationHandler.class});
  cons.setAccessible(true);
  return cons.newInstance(new Object[]{h});
}
```

### InvocationHandler

```java
public interface InvocationHandler{
  Object invoke(Object proxy, Method method, Object[] args) throws Throwable;
}
```

- `proxy`: the dynamic proxy object implementing the interface. Most often you don't need this object.
- `method`:  the method called on the interface the dynamic proxy implements. From the Method object you can obtain the method name, parameter types, return type, etc.
- `args`: the parameter values passed to the proxy when the method in the interface implemented was called.

### sun.misc.ProxyGenerator

Java的代理类就是通过`ProxyGenerator`来生成的，代理类生成之后是一串字节码字节数组，然后由我们提供的ClassLoader进行加载。如果我们想探究这个代理类的话，那么可以通过设置属性`sun.misc.ProxyGenerator.saveGeneratedFiles`为"ture"，那么就会在项目根目录下生成对应的`.class`字节码文件。

**设置属性**

```java
System.getProperties().setProperty("sun.misc.ProxyGenerator.saveGeneratedFiles", "true");
```

**代理类类名规则**

代理类的命名为：`com.sun.proxy.$ProxyN`，其中`N`是Proxy维护的一个自增的数字，每次新增生成代理类的时候其值就会自增。

### 代理类示例

**Action接口**

```java
public interface Action {
    void action(int animal, int move);
}
```

**生成代理类**

```java
public final class $Proxy0 extends Proxy implements Action {
  private static Method m1;
  private static Method m2;
  private static Method m3;
  private static Method m0;

  public $Proxy0(InvocationHandler var1) throws  {
    super(var1);
  }

  static {
    try {
      m0 = Class.forName("java.lang.Object").getMethod("hashCode");
      m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
      m2 = Class.forName("java.lang.Object").getMethod("toString");
      m3 = Class.forName("com.weibo.oasis.demo.proxy.Action").getMethod("action", Integer.TYPE, Integer.TYPE);
    } catch (NoSuchMethodException var2) {
      throw new NoSuchMethodError(var2.getMessage());
    } catch (ClassNotFoundException var3) {
      throw new NoClassDefFoundError(var3.getMessage());
    }
  }

  public final int hashCode() throws  {
    try {
      return (Integer)super.h.invoke(this, m0, (Object[])null);
    } catch (RuntimeException | Error var2) {
      throw var2;
    } catch (Throwable var3) {
      throw new UndeclaredThrowableException(var3);
    }
  }

  public final boolean equals(Object var1) throws  {
    try {
      return (Boolean)super.h.invoke(this, m1, new Object[]{var1});
    } catch (RuntimeException | Error var3) {
      throw var3;
    } catch (Throwable var4) {
      throw new UndeclaredThrowableException(var4);
    }
  }

  public final String toString() throws  {
    try {
      return (String)super.h.invoke(this, m2, (Object[])null);
    } catch (RuntimeException | Error var2) {
      throw var2;
    } catch (Throwable var3) {
      throw new UndeclaredThrowableException(var3);
    }
  }

  public final void action(int var1, int var2) throws  {
    try {
      super.h.invoke(this, m3, new Object[]{var1, var2});
    } catch (RuntimeException | Error var4) {
      throw var4;
    } catch (Throwable var5) {
      throw new UndeclaredThrowableException(var5);
    }
  }

}
```

**总结**

从代码可以看出，代理类继承自`java.reflect.Proxy`，除了实现接口Action里的方法之外，还实现了`Object`基础类的三个方法 **hashCode()**/**equals()**/**toString()**，这些方法都是直接调用的我们提供的InvocationHandler里的`invoke()`方法。

因此，我们提供的InvocationHandler除了需要实现接口对应的方法外，还需要实现 **hashCode()**/**equals()**/**toString()** 这三个方法。

- 代理类继承自`java.reflect.Proxy`基类，因为Java不支持多重继承，所以Java动态代理只支持代理接口，而不能代理类；
- InvocationHandler除了需要实现接口对应的方法外，还需要实现 **hashCode()**/**equals()**/**toString()** 这三个方法；
- `InvocationHandler.invoke()`方法虽然提供了proxy实例，但是不要调用proxy对象上的任何方法，避免循环调用导致死循环；

### MyBatis如何使用动态代理

**DefaultSqlSession**

```java
@Override
public <T> T getMapper(Class<T> type) {
  return configuration.getMapper(type, this);
}
```

我们通过`DefaultSqlSession.getMapper()`方法获取Mapper接口的实现类，这个实现类就是通过Java动态代理实现的。

**Configuration**

```java
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
  return mapperRegistry.getMapper(type, sqlSession);
}
```

**MapperRegistry**

```java
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
  final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
  if (mapperProxyFactory == null) {
    throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
  }
  try {
    return mapperProxyFactory.newInstance(sqlSession);
  } catch (Exception e) {
    throw new BindingException("Error getting mapper instance. Cause: " + e, e);
  }
}
```

**MapperProxyFactory**

```java
public class MapperProxyFactory<T> {

  private final Class<T> mapperInterface;
  private final Map<Method, MapperMethod> methodCache = new ConcurrentHashMap<>();

  public MapperProxyFactory(Class<T> mapperInterface) {
    this.mapperInterface = mapperInterface;
  }

  public Class<T> getMapperInterface() {
    return mapperInterface;
  }

  public Map<Method, MapperMethod> getMethodCache() {
    return methodCache;
  }

  @SuppressWarnings("unchecked")
  protected T newInstance(MapperProxy<T> mapperProxy) {
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
  }

  public T newInstance(SqlSession sqlSession) {
    final MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
  }

}
```

这里我们看到了`Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy)`，这里就是通过Java动态代理去生成的一个实现类。具体的实现细节就不细究了，这里只是看看Java动态代理的一个具体应用。