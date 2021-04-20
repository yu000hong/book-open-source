# TaskExecutor

`org.springframework.core.task`提供了任务执行的一些抽象，其中主要的三个接口：

- TaskExecutor
- AsyncTaskExecutor
- AsyncListenableTaskExecutor

### 接口描述

**TaskExecutor**

```java
public interface TaskExecutor extends Executor {
    @Override
    void execute(Runnable task);
}
```

`TaskExecutor`继承了Java中的线程池接口`java.util.concurrent.Executor`，提供了任务执行的抽象。

**AsyncTaskExecutor**

```java
public interface AsyncTaskExecutor extends TaskExecutor {
    //在各实现里，startTimeout意义不大，它本身也只是一个提示
    void execute(Runnable task, long startTimeout);
    Future<?> submit(Runnable task);
    Future<T> submit(Callable<T> task);
}
```

`AsyncTaskExecutor`提供了两个submit方法，通过返回的Future对提交的任务提供更多掌控。Future提供了对任务的取消操作，判断是否已经执行完成，任务是否已经取消，获取任务执行结果：

```java
public interfact Future<V> {
    boolean cancel(boolean mayInterruptIfRunning);
    boolean isCancelled();
    boolean isDone();
    V get() throws InterruptedException, ExecutionException;
    V get(long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException;
}
```

**AsyncListenableTaskExecutor**

```java
public interface AsyncListenableTaskExecutor extends AsyncTaskExecutor {
    ListenableFuture<?> submitListenable(Runnable task);
    ListenableFuture<T> submitListenable(Callable<T> task);
}
```

因为Java自带的`Future`本身获取执行结果是阻塞的，同时没法对Future进行任务编排，所以Spring提供了自己的可编排的异步Future：`ListenableFuture`。因此，`AsyncListenableTaskExecutor`提供了两个submit方法，通过返回`ListenableFuture`我们可以对异步任务进行编排，避免阻塞操作，达到真正的异步编程。

### 具体实现

```
TaskExecutor
   |---SyncTaskExecutor
   |---AsyncTaskExecutor
          |---AsyncListenableTaskExecutor
                 |---SimpleAsyncTaskExecutor
                 |---SimpleThreadPoolTaskExecutor
                 |---ThreadPoolTaskExecutor
                 |---ConcurrentTaskExecutor
                 |---ThreadPoolTaskScheduler
                 |---TaskExecutorAdaptor
```

**SyncTaskExecutor**

同步执行，就是在提交任务的线程执行任务。

**SimpleAsyncTaskExecutor**

每次都开启一个新线程来执行任务，任务执行完成则线程结束。如果开启限制(ConcurrencyLimit)，那么在线程数量达到限制数量的时候，提交任务会导致阻塞，避免无限制新建线程。

**ThreadPoolTaskExecutor**

底层使用`ThreadPoolExecutor`来提供线程池的功能！

**ThreadPoolTaskScheduler**

底层使用`ScheduledThreadPoolExecutor`来提供Scheduler的功能！

**TaskExecutorAdaptor**

一个适配器，将`java.util.concurrent.Executor`适配为`AsyncListenableTaskExecutor`。

**ConcurrentTaskExecutor**

> Autodetects a JSR-236 ManagedExecutorService in order to expose ManagedTask adapters for it, exposing a long-running hint based on SchedulingAwareRunnable and an identity name based on the given Runnable/Callable's toString().

### 异常类型

task包提供了两个异常类型：

- TaskRejectedException
- TaskTimeoutException

这两个异常都继承自`java.util.concurrent.RejectedExecutionException`，都是运行时异常！我们在提交任务的时候要考虑到有可能会抛出这两种异常，是否需要我们进行处理。

### 适配模式

task包提供了三个适配器类：

- TaskExecutorAdapter: Executor -> TaskExecutor
- ConcurrentExecutorAdaptor: TaskExecutor -> Executor
- ExecutorServiceAdaptor: TaskExecutor -> ExecutorService

### 任务装饰器

```java
public interface TaskDecorator {
    Runnable decorate(Runnable runnable);
}
```

