# AbstractQueuedSynchronizer

AbstractOwnableSynchronizer
     |---AbstractQueuedLongSynchronizer
     |---AbstractQueuedSynchronizer
                 |---ThreadPoolExecutor.Worker
                 |---CountDownLatch.Sync
                 |---ReentrantLock.Sync
                 |       |---ReentrantLock.FairSync
                 |       |---ReentrantLock.NonfairSync
                 |
                 |---ReentrantReadWriteLock.Sync
                 |       |---ReentrantReadWriteLock.FairSync
                 |       |---ReentrantReadWriteLock.NonfairSync
                 |
                 |---Semaphore.Sync
                         |---Semaphore.FairSync
                         |---Semaphore.NonfairSync


### 参考资料

[浅谈Java并发编程系列（八）—— LockSupport原理剖析](https://segmentfault.com/a/1190000008420938)

[What is the usage of the parameter of LockSupport.park(Object blocker)?](https://stackoverflow.com/questions/36939218/what-is-the-usage-of-the-parameter-of-locksupport-parkobject-blocker)

[CLH lock 原理及JAVA实现](https://www.cnblogs.com/shoshana-kong/p/10831502.html)

[Algorithms for Scalable Synchronization on Shared-Memory Multiprocessors](https://www.cs.rochester.edu/research/synchronization/pseudocode/ss.html)

[CLH Mutex - An alternative to the MCS Lock](http://concurrencyfreaks.blogspot.com/2014/05/exchg-mutex-alternative-to-mcs-lock.html)

[自旋锁队列CLH Lock简介](https://www.jianshu.com/p/f43e581976b9)

[CLH lock queue的原理解释及Java实现](https://zhuanlan.zhihu.com/p/161629590)

