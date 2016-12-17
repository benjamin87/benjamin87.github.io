---
layout: post
title:  "ConcurrentHashMap源码解读(size)-jdk8"
date:   2016-12-17 10:12:17 +0800
categories: tech
author: "Benjamin"
tags:
    - java
    - 并发
---

jdk8版本的 **ConcurrentHashMap** 在并发性上进行了诸多改进。在计算Map当前大小size的方法中，借鉴了jdk8新引入的类[LongAdder][LongAdder]中分而治之的求和方式。在并发密集的情况下，将总和值数拆分记录在数组中，最后再求和获得总和值。这种思路其实和ConcurrentHashMap本身的设计异曲同工。本文就先分析下 **LongAdder** 的源码与设计思路，接着解读ConcurrentHashMap的size方法便会迎刃而解了。

---

#### LongAdder

**LongAdder** 类位于包java.util.concurrent.atomic下。从包名可以看出，该类是一个注重并发的原子类，与之可比较的是原有的[AtomicLong][AtomicLong]类。

我们都知道，在AtomicLong的实现中，使用的是Unsafe类提供的CAS操作，以达到并发的正确性。在jdk8的改进中，该类中几乎所有的组合操作（即一个Get与一个其他操作的组合函数，如getAndUpdate、getAndAccumulate），都是采用了类似SpinLock的机制，在do-while语句中不停尝试CAS操作，直至操作成功。如下getAndUpdate函数源码：

```java
  // 增加AtomicLong中的当前值，并返回增加前的数值
  public final long getAndUpdate(LongUnaryOperator updateFunction) {
    long prev, next;
    do {
        prev = get();
        next = updateFunction.applyAsLong(prev);
    // 直到执行CAS操作成功，方可退出，否则一直尝试CAS操作
    } while (!compareAndSet(prev, next));
    return prev;
  }
```

与之相对，在LongAdder源码中，有如下的一段注释：

>Under low update contention, the two classes have similar characteristics. But under high contention, expected throughput of this class is significantly higher, at the expense of higher space consumption.

“在高并发场景下，LongAdder以空间为代价，能够获得比AtomicLong更高的性能优势。” 那么问题来了，LongAdder类内部又是如何实现的？使之能够和我们平时惯用的原子类中CAS+SpinLock的模式一较高下呢？

---

#### Cell

Cell定义在LongAdder父类Striped64中，具体定义如下：

```java
  /** 每个Cell可以看做是一个桶，当并发量高时，把总和值的不同部分存储在不同的Cell中 */
  @sun.misc.Contended static final class Cell {
        /** 借助volatile语义，实现并发读写Cell中的真实数据value字段 */
        volatile long value;
        Cell(long x) { value = x; }
        /** 设值操作仍然需要借助Unsafe类的CAS方法 */
        final boolean cas(long cmp, long val) {
            return UNSAFE.compareAndSwapLong(this, valueOffset, cmp, val);
        }
        // 以下省略部分代码
    }
```

**@sun.misc.Contended** 是jdk8中新引入的注解，主要目的是为了避免多个共享变量在同一缓存行中的[伪共享(false sharing)][false-sharing] 现象，获得更好的并发性能。

---

#### Striped64

了解分而治之设计思想中最基础的Cell数组结构，再来看下Striped64类。Striped64作为父类，实现了LongAdder核心逻辑，主要的数据字段如下：

```java
    /** 获取CPU个数，该数值决定了Cell数组的最大值 */
    static final int NCPU = Runtime.getRuntime().availableProcessors();

    /**
     * 核心数据结构Cell数组，总数值的分值分别存在每个cell中，每一个cell是保证并发正确性的最小单位
     */
    transient volatile Cell[] cells;

    /**
     * 总和值的获得方式为 base + 每个cell中分值之和
     * 在并发度较低的场景下，所有值都直接累加在base中
     */
    transient volatile long base;

    /**
     * 标识当前cell数组是否在初始化或扩容中的CAS标志位
     */
    transient volatile int cellsBusy;
```

Striped64类中除了对以上这些字段的CAS操作外，最主要的方法就是longAccumulate/doubleAccumulate实现增加操作，两个方法基本一致，以longAccumulate源码讲解如下：

```java
    final void longAccumulate(long x, LongBinaryOperator fn,
                              boolean wasUncontended) {
        int h;
        // 获取当前线程的probe值（与随机值有关）
        // 如果为0，则初始化当前线程probe随机值(hash)
        if ((h = getProbe()) == 0) {
            // 该静态方法会初始化当前线程所持有的随机值
            ThreadLocalRandom.current();
            // 获取生成后的probe值，用于选择cells数组下标元素
            h = getProbe();
            // 由于重新生成了probe，未冲突标志位设置为true
            wasUncontended = true;
        }
        // 当前线程的probe值，分配到cells数组下标元素不为空，则需要设置cell冲突标志位
        boolean collide = false;                // True if last slot nonempty
        // 直接进入死循环
        for (;;) {
            Cell[] as; Cell a; int n; long v;
            // cells数组已经被成功初始化，则放入对应的cell元素中
            if ((as = cells) != null && (n = as.length) > 0) {
                // n为2的次幂，n-1则为一个二进制低位全1的值，通过该值与当前线程probe求与，获得cells的下标元素
                // 该if语句对应的情况：当前线程probe值对应的cell数组下标处为null，则需要创建该cell元素
                if ((a = as[(n - 1) & h]) == null) {
                    // cellsBusy不为0表示cells数组不在初始化或者扩容状态下
                    if (cellsBusy == 0) {       // Try to attach new Cell
                        Cell r = new Cell(x);   // Optimistically create
                        // CAS设置cellsBusy，进入临界区，将创建的元素r放入cells数组中
                        if (cellsBusy == 0 && casCellsBusy()) {
                            boolean created = false;
                            try {               // Recheck under lock
                                Cell[] rs; int m, j;
                                if ((rs = cells) != null &&
                                    (m = rs.length) > 0 &&
                                    rs[j = (m - 1) & h] == null) {
                                    // 对应下标的cell为空，将新创建的cell放入对应下标位置，则累加操作成功
                                    rs[j] = r;
                                    created = true;
                                }
                            } finally {
                                // 释放临界区
                                cellsBusy = 0;
                            }
                            // 对应下标的cell为空，将新创建的cell放入对应下标位置，则累加操作成功
                            // 直接返回退出死循环
                            if (created)
                                break;
                            // 当前cell数组下标位置不为空，并发导致的不一致，重新进入死循环
                            continue;
                        }
                    }
                    collide = false;
                }
                // 获取了probe对应cells数组中的下标元素，发现不为空
                // 并且调用该函数前，调用方CAS操作也已经失败（已经发生竞争）
                else if (!wasUncontended)       // CAS already known to fail
                    // 设置未冲突标志位为true后，重新生成probe，进入死循环
                    wasUncontended = true;      // Continue after rehash
                // 使用CAS操作，在当前下标处的cell元素中执行累加
                else if (a.cas(v = a.value, ((fn == null) ? v + x :
                                             fn.applyAsLong(v, x))))
                    // 累加成功，则退出死循环
                    break;
                // cells数组元素足够大（不扩容），亦或者表在扩容过程中，已经过时
                // 重新生成probe后，进入死循环重试
                else if (n >= NCPU || cells != as)
                    collide = false;            // At max size or stale
                // 当前cells数组中下标元素不为空，则设置cell冲突标志位
                // 重新生成probe后，进入死循环重试
                // 该if语句避免了一冲突就扩容的情况
                else if (!collide)
                    collide = true;
                // 对Cell数组进行扩容，CAS设置cellsBusy进入临界区
                else if (cellsBusy == 0 && casCellsBusy()) {
                    try {
                        if (cells == as) {      // Expand table unless stale
                            Cell[] rs = new Cell[n << 1];
                            for (int i = 0; i < n; ++i)
                                rs[i] = as[i];
                            cells = rs;
                        }
                    } finally {
                        cellsBusy = 0;
                    }
                    collide = false;
                    continue;                   // Retry with expanded table
                }
                // 与ThreadLocalRandom中的方法一样
                // 为当前线程重新生成一个probe值，用于重新选择cells下标
                h = advanceProbe(h);
            }
            // 尝试初始化cells数组
            // 将cellsBusy变量从0设置为1，获取锁，进入初始化的临界区
            else if (cellsBusy == 0 && cells == as && casCellsBusy()) {
                boolean init = false;
                try {                           // Initialize table
                    if (cells == as) {
                        // cell数组大小必须为2的次幂
                        Cell[] rs = new Cell[2];
                        // 将需要累加的值x，放入cell数组元素中
                        // 选择的下标与当前线程的probe随机值有关
                        rs[h & 1] = new Cell(x);
                        cells = rs;
                        init = true;
                    }
                } finally {
                    cellsBusy = 0;
                }
                // 临界区中初始化成功，并且该x值被累加入cell中，则退出死循环
                if (init)
                    break;
            }
            // 没有cells数组，并且没有初始化的情况下，考虑直接累加在base变量中
            else if (casBase(v = base, ((fn == null) ? v + x :
                                        fn.applyAsLong(v, x))))
                // 如果直接累加在base中成功，则可直接退出死循环
                break;                         
        }
    }
```

具体的流程图整理如下：

![longAccumulate]({{ site.baseurl }}/img/20161217/1.jpg){: .center-image }

---

#### 再看LongAdder

解读完父类Striped64的具体实现细节后，再来看LongAdder类就简单很多了。该类主要的方法为add、sum，其内部就是委托给上述提及的父类实现即可。其中cells数组就是在父类中定义的核心数据结构。

```java
  public void add(long x) {
        Cell[] as; long b, v; int m; Cell a;
        // 直接尝试CAS操作，累加x在base中
        if ((as = cells) != null || !casBase(b = base, b + x)) {
            boolean uncontended = true;
            if (as == null || (m = as.length - 1) < 0 ||
                // 获取cells数组下标处为空，则调用父类方法创建新cell
                (a = as[getProbe() & m]) == null ||
                // 当前cells数组下标处不为空，则CAS操作累加在该cell节点上
                // 失败则设置uncontended冲突标志位为false
                !(uncontended = a.cas(v = a.value, v + x)))
                longAccumulate(x, null, uncontended);
        }
    }

  public long sum() {
     Cell[] as = cells; Cell a;
     // 首先初始sum为base值
     long sum = base;
     if (as != null) {
         for (int i = 0; i < as.length; ++i) {
             if ((a = as[i]) != null)
                 // cells数组不为空，则累加该数组每个cell节点的值
                 sum += a.value;
         }
     }
     return sum;
 }
```

---

#### size方法

完全理清LongAdder内部实现后，再来看ConcurrentHashMap的size方法，就会豁然开朗。借鉴LongAdder的思想，ConcurrentHashMap内部也定义了相应的Cell——**CounterCell**

```java
   // 与Cell定义一模一样
   @sun.misc.Contended static final class CounterCell {
        volatile long value;
        CounterCell(long x) { value = x; }
    }

    final long sumCount() {
        CounterCell[] as = counterCells; CounterCell a;
        // 首先初始sum为base值
        long sum = baseCount;
        if (as != null) {
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null)
                    // cells数组不为空，则累加该数组每个cell节点的值
                    sum += a.value;
            }
        }
        return sum;
    }
```

---

#### 总结

通过解读LongAdder和ConcurrentHashMap源码中对size的处理，无法不惊叹于作者在并发性能上的缜密思考。利用空间的分散性降低了并发性上的冲突概率，利用哈希的思想，以空间换取性能，实在是令人拍案叫距。


[LongAdder]: http://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/LongAdder.html
[AtomicLong]:https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/AtomicLong.html
[false-sharing]:http://ifeve.com/falsesharing/
