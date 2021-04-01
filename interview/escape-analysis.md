# 逃逸分析

Escape analysis examines a variable, looks at where it's used and detects whether it's used outside a certain scope. If it doesn't escape that scope, we know it's a local variable, and we can examine it more closely. Variables used only in a limited scope allow for many optimizations.

使用逃逸分析，编译器可以做如下优化：

- 锁消除：如果一个对象被发现只能从一个线程被访问到，那么对于这个对象的操作可以不考虑同步。
- 栈分配：如果一个对象在子程序中被分配，要使指向该对象的指针永远不会逃逸，对象可能是栈分配的候选，而不是堆分配。
- 标量替换：有的对象可能不需要作为一个连续的内存结构存在也可以被访问到，那么对象的部分（或全部）可以不存储在内存，而是存储在CPU寄存器中。

**逃逸分析并不成熟**

关于逃逸分析的论文在1999年就已经发表了，但直到JDK 1.6才有实现，而且这项技术到如今也并不是十分成熟的。**其根本原因就是无法保证逃逸分析的性能消耗一定能高于他的消耗**。虽然经过逃逸分析可以做标量替换、栈上分配、和锁消除。但是逃逸分析自身也是需要进行一系列复杂的分析的，这其实也是一个相对耗时的过程。

### What is the stack and why is it faster than the heap?



### Lock elision

One of these optimizations is lock elision.

In most cases, it recognizes that the critical section is used by a single method. So it knows for sure there's one and only one thread accessing it. In other words: the `synchronized` statement is superfluous. It can safely be omitted.

```java
import org.openjdk.jmh.annotations.*;

@Warmup(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Measurement(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Fork(3)
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Benchmark)
public class LockElision {

    int x;

    @Benchmark
    public void baseline() {
        x++;
    }

    @Benchmark
    public void locked() {
        synchronized (new Object()) {
            x++;
        }
    }
}
```

If we supply JVM flag `-XX:-EliminateLocks`, or we disable EA with `-XX:-DoEscapeAnalysis` (that breaks every optimization that depends on EA, including lock elision), then locked counters would balloon up.

### 参考资源

[Escape Analysis in Java](https://www.beyondjava.net/escape-analysis-java)

[JVM Anatomy Quark #19: Lock Elision](https://shipilev.net/jvm/anatomy-quarks/19-lock-elision/)