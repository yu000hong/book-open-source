# GC调优

### 元空间

MetaspaceSize is the value when a Full GC will be triggered, its not initial space nor maximum space. The default value for it is 20MB (at least on jdk-8 to jdk-13):

```
java -XX:+PrintFlagsFinal -version | grep MetaspaceSize
```

will produce:

```
size_t MetaspaceSize = 21807104 // 20MB
```

This means that when Metaspace reaches 20MB a Full GC will be triggered and if you can avoid it, that would be good.

设置元空间大小：

```
-XX:MetaspaceSize=100M
```


### 一些概念

**Used memory**

> Used memory(**jvm.memory.used**): It is the current amount of memory in use.

**Committed memory**

> Committed memory(**jvm.memory.committed**): It is the amount of memory that is guaranteed to be available for use by the Java virtual machine. The amount of committed memory can easily change over time. The Java virtual machine may release memory to the system and committed memory could be less than initial memory. Committed memory will always be greater than or equal to the used memory.

**Max memory**

> Max memory(**jvm.memory.max**): It is stated as the maximum amount of memory that can be used for memory management. You may observe that many a times its value is undefined. The maximum amount of memory may change over time if defined. In every case where max memory is defined, the amount of used and committed memory will always be less than or equal to max.

### 垃圾回收器名称

**Young generation collectors**

> **Copy (enabled with -XX:+UseSerialGC)**
> 
> the serial copy collector, uses one thread to copy surviving objects from Eden to Survivor spaces and between Survivor spaces until it decides they've been there long enough, at which point it copies them into the old generation.

> **PS Scavenge (enabled with -XX:+UseParallelGC)**
>
> the parallel scavenge collector, like the Copy collector, but uses multiple threads in parallel and has some knowledge of how the old generation is collected (essentially written to work with the serial and PS old gen collectors).

> **ParNew (enabled with -XX:+UseParNewGC)**
>
> the parallel copy collector, like the Copy collector, but uses multiple threads in parallel and has an internal 'callback' that allows an old generation collector to operate on the objects it collects (really written to work with the concurrent collector).

> **G1 Young Generation (enabled with -XX:+UseG1GC)**
>
> the garbage first collector, uses the 'Garbage First' algorithm which splits up the heap into lots of smaller spaces, but these are still separated into Eden and Survivor spaces in the young generation for G1.

**Old generation collectors**

> **MarkSweepCompact (enabled with -XX:+UseSerialGC)**
>
> the serial mark-sweep collector, the daddy of them all, uses a serial (one thread) full mark-sweep garbage collection algorithm, with optional compaction.

> **PS MarkSweep (enabled with -XX:+UseParallelOldGC)**
>
> the parallel scavenge mark-sweep collector, parallelised version (i.e. uses multiple threads) of the MarkSweepCompact.

> **ConcurrentMarkSweep (enabled with -XX:+UseConcMarkSweepGC)**
>
> the concurrent collector, a garbage collection algorithm that attempts to do most of the garbage collection work in the background without stopping application threads while it works (there are still phases where it has to stop application threads, but these phases are attempted to be kept to a minimum). Note if the concurrent collector fails to keep up with the garbage, it fails over to the serial MarkSweepCompact collector for (just) the next GC.

> **G1 Mixed Generation (enabled with -XX:+UseG1GC)**
> the garbage first collector, uses the 'Garbage First' algorithm which splits up the heap into lots of smaller spaces.


### 参考资料

[HotSpot Virtual Machine Garbage Collection Tuning Guide](https://docs.oracle.com/javase/10/gctuning/toc.htm)

[Oracle JVM Garbage Collectors Available From JDK 1.7.0_04 And After](http://www.fasterj.com/articles/oraclecollectors1.shtml)