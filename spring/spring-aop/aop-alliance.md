# AOP Alliance

> The AOP Alliance project is a joint open-source project between several software engineering people who are interested in AOP and Java. AOP Alliance intends to facilitate and standardize the use of AOP to enhance existing middleware environments (such as J2EE), or development environements (e.g. JBuilder, Eclipse). The AOP Alliance also aims to ensure interoperability between Java/J2EE AOP implementations to build a larger AOP community.

官网：[aopalliance.sourceforge.net](http://aopalliance.sourceforge.net/)

### Joinpoint

`Joinpoint`指连接点，是指我们在代码的什么地方可以插入我们的横切逻辑，即上面讲的通知`Advice`。

我们看看其源代码：

```java
public interface Joinpoint {
    AccessibleObject getStaticPart();
    Object getThis();
    Object proceed();
}
```

`getStaticPart()`：TODO

`getThis()`：TODO

`proceed()`：这个方法表示的就是我们连接点处被拦截的执行逻辑。我们可以在这个拦截处的前面执行横切逻辑，就是Spring AOP的`BeforeAdvice`；我们可以在这个拦截处的后面执行横切逻辑，就是Spring AOP的`AfterAdvice`；如果我们捕获到异常之后执行横切逻辑，就是Spring AOP的`ThrowsAdvice`；你也可以根据你的需要去实现`Advice`，实现自己的横切逻辑。

**连接点类层次结构**

```
Joinpoint
   |---FieldAccess
   |---Invocation
          |---MethodInvocation
          |---ConstructorInvocation
```

根据连接点的类型不同，有三个不同的子类：

- FieldAccess：**字段访问**类型连接点，表示的一个字段的读取或写入过程
- MethodInvocation：**方法调用**类型连接点，表示的一个方法的调用过程
- ConstructorInvocation：**构造器**类型连接点，表示的是对象构建过程

### Advice

通知，AOP在连接点要执行的代码。

```
                                Advice
                                   |
                              Interceptor
                                   |
        |--------------------------|----------------------|
        |                          |                      |
ConstructorInterceptor      MethodInterceptor       FieldInterceptor
```

`Advice`和`Interceptor`都是标志类型接口，空接口。

根据连接点的类型，AOP联盟定义了三种对应类型的拦截器：

- ConstructorInterceptor：构造器拦截器
- MethodInterceptor：方法调用拦截器
- FieldInterceptor：字段访问拦截器

```java
public interface ConstructorInterceptor {
    Object	construct(ConstructorInvocation invocation);
}
public interface MethodInterceptor {
    Object	invoke(MethodInvocation invocation);
}
public interface FieldInterceptor {
    Object	get(FieldAccess fieldRead);
    Object	set(FieldAccess fieldWrite);
}
```

我们可以通过实现这些拦截器来织入我们的横切逻辑！

**⚠️注意**：Spring AOP框架只支持方法调用拦截器`MethodInterceptor`。
