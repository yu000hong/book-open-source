# Java引用

四种类型：
- StrongReference
- SoftReference
- WeakReference
- PhantomReference

StrongReference > SoftReference > WeakReference > PhantomReference

> The package `java.lang.ref` contains classes which can wrap any arbitrary object and are used as an indication to the garbage collector that how the underlying object should be garbage collected. For a Java program, the main advantage of using these classes is to **know when an object has been garbage collected**.

- `WeakReference`: is used to hold an object which will become eligible for the garbage collection as soon as it is not reachable by the program.
- `SoftReference`: lives longer, it will only be garbage collected before an OutOfMemoryError is thrown.

- 强引用没有对应的类型表示，也就是说强引用是普遍存在的，如Object object = new Object();。
- 软引用、弱引用和虚引用都是java.lang.ref.Reference的直接子类。
- 直到JDK11为止，只存在四种引用，这些引用是由JVM创建，因此直接继承java.lang.ref.Reference创建自定义的引用类型是无效的，但是可以直接继承已经存在的引用类型，如java.lang.ref.Cleaner就是继承自java.lang.ref.PhantomReference。


### finalize()不推荐使用，Java11已废弃

基于如下两个原因，`finalize()`已不推荐使用，将被废弃：

- 执行次数
- 复活问题

**执行次数**

> Java only invokes the finalizer at most one time. As we may recall, there is no guarantee that the finalizer is ever invoked.

`finalize()`方法最多执行一次，有可能不会执行（比如程序退出，JVM里面的对象就不会再执行finalize()方法）。

**复活问题**

如果在执行`finalize()`方法的时候，我们刻意或不经意间重新引用了当前对象，那么该对象马上又可达，即复活了。如：

```java
public class Immortal {

    private static final Set<Immortal> immortals = new HashSet<>();

    @Override
    protected void finalize() throws Throwable {
        System.out.println(Immortal.class.getSimpleName() + "::finalize for " + this);
        immortals.add(this); // Resurrect the object by creating a new reference 
    }

}
```