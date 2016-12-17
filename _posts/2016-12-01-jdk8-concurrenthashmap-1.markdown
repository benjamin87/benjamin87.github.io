---
layout: post
title:  "ConcurrentHashMap源码解读(put/transfer/get)-jdk8"
date:   2016-12-01 22:25:49 +0800
categories: tech
author: "Benjamin"
tags:
    - java
    - 并发
---

在java的集合家族中， **ConcurrentHashMap** 当之无愧是支持并发最好的键值对（Map）集合。在日常编码中，出场率也相当之高。在jdk8中，集合类 ConcurrentHashMap 经 *Doug Lea* 大师之手，借助volatile语义以及CAS操作进行优化，使得该集合类更好地发挥出了并发的优势。与jdk7中相比，在原有段锁（Segment）的基础上，引入了数组＋链表＋红黑树的存储模型，在查询效率上花费了不少心思。

>本文作为源码解读笔记，逐一分析关键函数，初探下jdk8版本的ConcurrentHashMap在并发与性能设计上的精妙之处。

![clazz-uml]({{ site.baseurl }}/img/20161201/1.jpg){: .center-image }

#### 基础数据结构

ConcurrentHashMap内存存储结构图大致如下：

![data-structure]({{ site.baseurl }}/img/20161201/2.jpg){: .center-image }

ConcurrentHashMap部分成员变量/常量分析如下（详见注释）：

```java
   /**
    * 真正存储Node数据（桶首）节点的数组table
    * 所有Node节点根据hash分桶存储
    * table数组中存储的是所有桶（bin）的首节点
    * hash值相同的节点以链表形式分装在桶中
    * 当一个桶中节点数达到8个时，转换为红黑树，提高查询效率
    */
   transient volatile Node<K,V>[] table;

   /**
    *  只在扩容时使用
    */
   private transient volatile Node<K,V>[] nextTable;

   /**
    *  重要控制变量
    *  根据变量的数值不同，类实例处于不同阶段
    *  1. = -1 : 正在初始化
    *  2. < -1 : 正在扩容，数值为 -(1 + 参与扩容的线程数)
    *  3. = 0  : 创建时初始为0
    *  4. > 0  : 下一次扩容的大小
    */
   private transient volatile int sizeCtl;

   /**
    *  table的最大容量，必须为2次幂形式
    */
   private static final int MAXIMUM_CAPACITY = 1 << 30;

   /**
    *  table的默认初始容量，必须为2次幂形式
    */
   private static final int DEFAULT_CAPACITY = 16;

   /**
    *  table的负载因子，当前节点数量超过 n * LOAD_FACTOR，执行扩容
    *  位操作表达式为 n - (n >>> 2)
    */
   private static final float LOAD_FACTOR = 0.75f;

   /**
    *  针对每个桶（bin），链表转换为红黑树的节点数阈值
    */
   static final int TREEIFY_THRESHOLD = 8;

   /**
    *  针对每个桶（bin），红黑树退化为链表的节点数阈值
    */
   static final int UNTREEIFY_THRESHOLD = 6;

   /**
    *  在扩容中，参与的单个线程允许处理的最少table桶首节点个数
    *  虽然适当添加线程，会使得整个扩容过程变快，但需要考虑多线程内存同时分配的问题
    */
   private static final int MIN_TRANSFER_STRIDE = 16;
```

---

Node<K,V>节点是ConcurrentHashMap存储数据的最基本结构。一个数据mapping节点中，存储4个变量：当前节点hash值、节点的key值、节点的value值、指向下一个节点的指针next。其中在子类中的hash可以为负数，具有特殊的并发处理意义，后文会解释。除了具有特殊意义的子类，Node中的key和val不允许为null。

```java
  /**
  * 基础的键值对节点
  */
  static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        volatile V val;             // 申明为volatile
        volatile Node<K,V> next;    // 申明为volatile

        // 此处省略部分成员函数

        Node<K,V> find(int h, Object k) {
              Node<K,V> e = this;
              if (k != null) {
                  do {
                      K ek;
                      if (e.hash == h &&
                          ((ek = e.key) == k || (ek != null && k.equals(ek))))
                          return e;
                  /**
                  *  以链表形式查找桶中下一个Node信息
                  *  当转换为subclass红黑树节点TreeNode
                  *  则使用TreeNode中的find进行查询操作
                  */
                  } while ((e = e.next) != null);
              }
              return null;
          }
  }  
```

---

在真正阅读 ConcurrentHashMap 源码之前，我们简单复习下关于volatile和CAS的概念，这样才能更好地帮助我们理解源码中的关键方法。

#### volatile语义

java提供的关键字volatile是最轻量级的同步机制。当定义一个变量为volatile时，它就具备了三层语义：
- 可见性（Visibility）：在多线程环境下，一个变量的写操作总是对其后的读取线程可见
- 原子性（Atomicity）：volatile的读/写操作具有原子性
- 有序性（Ordering）：禁止指令的重排序优化，JVM会通过插入内存屏障（Memory Barrier）指令来保证

就同步性能而言，大多数场景下volatile的总开销是要比锁低的。在ConcurrentHashMap的源码中，我们能看到频繁的volatile变量读取与写入。

```java
   /**
   *  读取Table数组表中的Node元素
   */
   static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
       // volatile读
       return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
   }

   /**
   *  赋值Table数组表中的Node元素
   */
   static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {
       // volatile写
       U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);
   }
```

这两个函数在ConcurrentHashMap中使用非常频繁。对数组表中Node，最基础的读取与赋值操作，都使用了Unsafe类中附带volatile语义的操作。这正是上层逻辑函数中能够尽可能去锁化的基石所在。

#### CAS操作

>CAS:Compare and Swap

```java
   /**
   *  利用CAS赋值Table数组表中的Node元素
   *  与volatile的setTabAt操作不同，setTabAt操作一定会成功，
   *  但casTabAt由于期望一个原有值expected，可能会失败
   */
   static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                        Node<K,V> c, Node<K,V> v) {
        return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
    }
```

CAS一般被理解为原子操作。在java中，正是利用了处理器的CMPXCHG（intel）指令实现CAS操作。CAS需要接受原有期望值expected以及想要修改的新值x，只有在原有期望值与当前值相等时才会更新为x，否则为失败。在ConcurrentHashMap的方法中，大量使用CAS获取/修改互斥量，以达到多线程并发环境下的正确性。

---

#### 源码分析之putVal方法

putVal是将一个新key-value mapping插入到当前ConcurrentHashMap的关键方法。

此方法的具体流程如下图：

![put-val]({{ site.baseurl }}/img/20161201/3.jpg){: .center-image }

putVal源码分析如下（详见注释）：

```java
  final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        // 计算新节点的hash值
        int hash = spread(key.hashCode());
        int binCount = 0;
        // 获取当前table，进入死循环
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            // 当前table为空，则进行初始化，然后重新进入死循环
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
            // hash值对应table数组中的桶首节点为空，则直接cas插入新节点为桶首节点
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    // 直接设置为桶首节点成功，退出死循环（出口之一）
                    break;                   
            }
            // 当前桶首节点正在特殊的扩容状态下，当前线程尝试参与扩容
            // 然后重新进入死循环
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            // 通过桶首节点，将新节点加入table
            else {
                V oldVal = null;
                // 获取桶首节点实例对象锁，进入临界区进行添加操作
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        // 桶首节点hash值>0，表示为链表
                        if (fh >= 0) {
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                // key已经存在，则替换
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                // key不存在，则插入在链表尾部
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        // 桶首节点为Node子类型TreeBin，表示为红黑树
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            // 调用putTreeVal方法，插入新值
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                // key已经存在，则替换
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
                if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD)
                        // 插入新节点后，达到链表转换红黑树阈值，则执行转换操作
                        treeifyBin(tab, i);
                    // 退出死循环（出口之二）
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        // 更新计算count时的base和counterCells数组
        // 检查是否有扩容需求，如是，则执行扩容
        addCount(1L, binCount);
        return null;
    }
```

该流程中，可以细细品味的环节有：
- 初始化方法 **initTable**
- 扩容方法 **transfer** (在多线程扩容方法 **helpTransfer** 中被调用)

---

#### 源码分析之初始化initTable方法

initTable方法允许多线程同时进入，但只有一个线程可以完成table的初始化，其他线程都会通过yield方法让出cpu。

```java
  private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
            if ((sc = sizeCtl) < 0)
                // 前文提及sizeCtl是重要的控制变量
                // sizeCtl = -1 表示正在初始化
                // 已经有其他线程在执行初始化，则主动让出cpu
                Thread.yield();
            // 利用CAS操作设置sizeCtl为-1
            // 设置成功表示当前线程为执行初始化的唯一线程
            // 此处进入临界区
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    // 由于让出cpu的线程也会后续进入该临界区
                    // 需要进行再次确认table是否为null
                    if ((tab = table) == null || tab.length == 0) {
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                        // 真正初始化，即分配Node数组
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                        // 默认负载为0.75
                        sc = n - (n >>> 2);
                    }
                } finally {
                    sizeCtl = sc;
                }
                // 退出死循环的唯一出口
                break;
            }
        }
        return tab;
    }
```

---

#### 源码分析之扩容transfer方法

扩容transfer方法是一个设计极为精巧的方法。通过互斥读写ForwardingNode，多线程可以协同完成扩容任务。

![transfer]({{ site.baseurl }}/img/20161201/4.jpg){: .center-image }

transfer源码分析如下（详见注释）：

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
      int n = tab.length, stride;
      if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
          // 单个线程允许处理的最少table桶首节点个数
          // 即每个线程的处理任务量
          stride = MIN_TRANSFER_STRIDE;
      // nextTab为扩容中的临时table
      if (nextTab == null) {
          try {
              @SuppressWarnings("unchecked")
              // 扩容后的容量为当前的2倍
              Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
              nextTab = nt;
          } catch (Throwable ex) {
              sizeCtl = Integer.MAX_VALUE;
              return;
          }
          nextTable = nextTab;
          // transferIndex为扩容复制过程中的桶首节点遍历索引
          // 所以从n开始，表示从后向前遍历
          transferIndex = n;
      }
      int nextn = nextTab.length;
      // ForwardingNode是Node节点的直接子类，是扩容过程中的特殊桶首节点
      // 该类中没有key,value,next
      // hash值为特定的-1
      // 附加Node<K,V>[] nextTable变量指向扩容中的nextTab
      // 在find方法中，将扩容中的查询操作导入到nextTab上
      ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
      boolean advance = true;
      boolean finishing = false;
      for (int i = 0, bound = 0;;) {
          Node<K,V> f; int fh;
          while (advance) {
              int nextIndex, nextBound;
              if (--i >= bound || finishing)
                  advance = false;
              // transferIndex = 0表示table中所有数组元素都已经有其他线程负责扩容
              else if ((nextIndex = transferIndex) <= 0) {
                  i = -1;
                  advance = false;
              }
              // 尝试更新transferIndex，获取当前线程执行扩容复制的索引区间
              // 更新成功，则当前线程负责完成索引为(nextBound，nextIndex)之间的桶首节点扩容
              else if (U.compareAndSwapInt
                       (this, TRANSFERINDEX, nextIndex,
                        nextBound = (nextIndex > stride ?
                                     nextIndex - stride : 0))) {
                  bound = nextBound;
                  i = nextIndex - 1;
                  advance = false;
              }
          }
          if (i < 0 || i >= n || i + n >= nextn) {
              int sc;
              if (finishing) {
                  nextTable = null;
                  table = nextTab;
                  // 扩容成功，设置新sizeCtl，仍然为总大小的0.75
                  sizeCtl = (n << 1) - (n >>> 1);
                  return;
              }
              if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                finishing = advance = true;
                i = n;
              }
          }
          // 当前table节点为空，不需要复制，直接放入ForwardingNode
          else if ((f = tabAt(tab, i)) == null)
              advance = casTabAt(tab, i, null, fwd);
          // 当前table节点已经是ForwardingNode
          // 表示已经被其他线程处理了，则直接往前遍历
          // 通过CAS读写ForwardingNode节点状态，达到多线程互斥处理
          else if ((fh = f.hash) == MOVED)
              advance = true;
          // 此处开始执行真正的扩容操作
          else {
            // 锁住当前桶首节点
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;
                    // 链表节点复制
                    if (fh >= 0) {
                        int runBit = fh & n;
                        Node<K,V> lastRun = f;
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        // 扩容成功后，设置ForwardingNode节点
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                    // 红黑树节点复制
                    else if (f instanceof TreeBin) {
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                            (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                            (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        // 扩容成功后，设置ForwardingNode节点
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                }
            }
         }
      }
    }
```
---

#### 源码分析之get方法

相比之下，获取方法get要简单很多。唯一需要注意的是，即使在扩容情况下，get操作也能正确执行。

```java
    public V get(Object key) {
        Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
        //  计算当前key值的hash值
        int h = spread(key.hashCode());
        if ((tab = table) != null && (n = tab.length) > 0 &&
            // 唯一一处volatile读操作
            (e = tabAt(tab, (n - 1) & h)) != null) {
            // 判断是否就是桶首节点
            if ((eh = e.hash) == h) {
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                    return e.val;
            }
            // hash为负表示是扩容中的ForwardingNode节点
            // 直接调用ForwardingNode的find方法(可以是代理到扩容中的nextTable)
            else if (eh < 0)
                return (p = e.find(h, key)) != null ? p.val : null;
            // 通过next指针，逐一查找
            while ((e = e.next) != null) {
                if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                    return e.val;
            }
        }
        return null;
    }
```
---

#### 总结

通过分析ConcurrentHashMap中put、get、transfer方法，我们可以看到在jdk8下，该集合类在并发性上做出了诸多优化。volatile语义提供更细颗粒度的轻量级锁，使得多线程可以(几乎)同时读写实例中的关键量，正确理解当前类所处的状态，进入对应if语句中执行相关逻辑。通过学习这些关键方法中的并发处理，我们可以体会更多大师笔下的精妙设计。醍醐灌顶，茅塞顿开，岂不美哉？
