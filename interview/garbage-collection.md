# Garbage Collection

三种方式：
- Minor GC: cleaning the Young space
- Major GC: cleaning the Old space
- Full GC: cleaning the entire heap - both the Young and Old space

> Unfortunately it is a bit more complex and confusing. To start with – many Major GCs are triggered by Minor GCs, so separating the two is impossible in many cases. On the other hand – modern garbage collection algorithms like G1 perform partial garbage cleaning so, again, using the term ‘cleaning’ is only partially correct.

> HotSpot VM 的Minor GC全都使用Copying算法，差别只是串行还是并行而已。Copying算法的特征之一就是它的开销只跟活对象的多少（live data set）有关系，而跟它所管理的堆空间的大小没关系。

问题列表：
- 什么时候触发Minor GC？
- Minor GC 整个过程是怎样的？
- Minor GC 用的什么垃圾回收算法？复制整理算法？为什么不用标记清除算法？
- 为什么需要Survivor内存区？

注意事项：
- 特别要注意系统里面的短期存活的大对象(short-lived large objects)


### Dynamic Object Age

The JVM does not require that the age of an object must be 15 so that it can be promoted to the old generation. If objects of the same age occupy more than half of the Survivor space, objects of greater than or equal to this age can be directly promoted to the old generation without waiting be "old enough."

### 参考资源

[How Does Garbage Collection Work in Java?](https://www.alibabacloud.com/blog/how-does-garbage-collection-work-in-java_595387)

[Java Garbage Collection handbook](https://plumbr.io/java-garbage-collection-handbook)

[JVM GC遍历一次新生代所有对象是否可达需要多久？- RednaxelaFX 的回答 - 知乎](https://www.zhihu.com/question/33210180/answer/56348818)

[并发垃圾收集器（CMS）为什么没有采用标记-整理算法来实现？](https://hllvm-group.iteye.com/group/topic/38223#post-248757)

[java的gc为什么要分代？](https://www.zhihu.com/question/53613423/answer/135743258)

[有关 Copying GC 的疑问？- RednaxelaFX 的回答 - 知乎](https://www.zhihu.com/question/42181722/answer/93871206)

[关于HotSpot VM的Serial GC中的minor GC的“简单”讲解](https://hllvm-group.iteye.com/group/topic/39376#post-257329)

[Java中什么样的对象才能作为gc root，gc roots有哪些呢？](https://www.zhihu.com/question/50381439)