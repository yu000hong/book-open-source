# Executor

Executor接口包含如下方法：

继承关系：

```
Executor
   |---CachingExecutor
   |---BaseExecutor
            |---SimpleExecutor
            |---ReuseExecutor
            |---BatchExecutor
            |---ResultLoaderMap.ClosedExecutor
```


### ExecutorType

枚举类：

- SIMPLE
- REUSE
- BATCH

### BaseExecutor

BaseExecutor是SimpleExecutor/ReuseExecutor/BatchExecutor的父类，提供了一些基础功能，如：缓存执行结果，这其实就是MyBatis的一级缓存！

在每次执行`update()`/`commit()`/`rollback()`方法的时候，都会执行`clearLocalCache()`方法来清理一级缓存。

**先清理缓存再操作数据库，还是先操作数据库再清理缓存？** MyBatis给出了它的答案：先清缓存再操作数据库！

```java
public int update(MappedStatement ms, Object parameter) throws SQLException {
    if (closed) {
        throw new ExecutorException("Executor was closed.");
    }
    clearLocalCache();
    return doUpdate(ms, parameter);
}
```

### SimpleExecutor

不支持批量操作，不支持对JDBC Statement的重用。

### ReuseExecutor

SimpleExecutor每次执行完之后都会调用close()方法关闭JDBC Statement对象，而ReuseExecutor会重用Statement对象。

我们看看`doFlushStatements()`方法的代码：

```java
public List<BatchResult> doFlushStatements(boolean isRollback) {
    for (Statement stmt : statementMap.values()) {
        closeStatement(stmt);
    }
    statementMap.clear();
    return Collections.emptyList();
}
```

从代码可以看出，ReuseExecutor是不支持批量操作的，但是在`doFlushStatements()`中完成了对缓存Statement的关闭和清理。

### BatchExecutor

BatchExecutor支持批量更新操作，其他的方法(doQuery/doQueryCursor)和SimpleExecutor的实现基本一致，但是在这两个query方法里会执行一次`doFlushStatements()`方法，保证之前的批量更新操作都完成了。

### CachingExecutor

CachingExecutor主要是支持MyBatis二级缓存，它没有继承BaseExecutor，而是使用委托的方式内部使用了一个BaseExecutor子类来做真正的数据库操作。

MyBatis一级缓存是BaseExecutor带来的，因此始终提供。而二级缓存是可以由我们使用者进行配置的，如果我们在使用MyBatis配置了二级缓存，那么最终会采用CachingExecutor。