---
title: CountdownLatch和CyclicBarrier和ConcurrentHashMap
date: 2021-11-30 22:23:45
tags: Java并发
categories: java基础
---

# **CountdownLatch**

用来进行线程同步协作，等待所有线程完成倒计时。
其中构造参数用来初始化等待计数值，await() 用来等待计数归零，countDown() 用来让计数减一

```java
 CountDownLatch latch = new CountDownLatch(3);
 new Thread(() -> {
 log.debug("begin...");
 sleep(1);
 latch.countDown();
 log.debug("end...{}", latch.getCount());
 }).start();
 new Thread(() -> {
 log.debug("begin...");
 sleep(2);
 latch.countDown();
 log.debug("end...{}", latch.getCount());
 }).start();
 new Thread(() -> {
 log.debug("begin...");
 sleep(1.5);
 latch.countDown();
 log.debug("end...{}", latch.getCount());
 }).start();
 log.debug("waiting...");
 latch.await();
 log.debug("wait end...");
}
```

输出

```java
18:44:00.778 c.TestCountDownLatch [main] - waiting... 
18:44:00.778 c.TestCountDownLatch [Thread-2] - begin... 
18:44:00.778 c.TestCountDownLatch [Thread-0] - begin... 
18:44:00.778 c.TestCountDownLatch [Thread-1] - begin... 
18:44:01.782 c.TestCountDownLatch [Thread-0] - end...2 
18:44:02.283 c.TestCountDownLatch [Thread-2] - end...1 
18:44:02.782 c.TestCountDownLatch [Thread-1] - end...0 
18:44:02.782 c.TestCountDownLatch [main] - wait end...
```

可以配合线程池使用，改进如下

```java
public static void main(String[] args) throws InterruptedException {
 CountDownLatch latch = new CountDownLatch(3);
 ExecutorService service = Executors.newFixedThreadPool(4);
 service.submit(() -> {
 log.debug("begin...");
 sleep(1);
 latch.countDown();
 log.debug("end...{}", latch.getCount());
 });
 service.submit(() -> {
 log.debug("begin...");
 sleep(1.5);
 latch.countDown();
 log.debug("end...{}", latch.getCount());
 });
 service.submit(() -> {
 log.debug("begin...");
 sleep(2);
 latch.countDown();
 log.debug("end...{}", latch.getCount());
 });
 service.submit(()->{
 try {
 log.debug("waiting...");
 latch.await();
 log.debug("wait end...");
 } catch (InterruptedException e) {
 e.printStackTrace();
 }
 });
}
```

输出

```java
18:52:25.831 c.TestCountDownLatch [pool-1-thread-3] - begin... 
18:52:25.831 c.TestCountDownLatch [pool-1-thread-1] - begin... 
18:52:25.831 c.TestCountDownLatch [pool-1-thread-2] - begin... 
18:52:25.831 c.TestCountDownLatch [pool-1-thread-4] - waiting... 
18:52:26.835 c.TestCountDownLatch [pool-1-thread-1] - end...2 
18:52:27.335 c.TestCountDownLatch [pool-1-thread-2] - end...1 
18:52:27.835 c.TestCountDownLatch [pool-1-thread-3] - end...0 
18:52:27.835 c.TestCountDownLatch [pool-1-thread-4] - wait end...
```

# **应用之同步等待多线程准备完毕**

```java
AtomicInteger num = new AtomicInteger(0);
ExecutorService service = Executors.newFixedThreadPool(10, (r) -> {
     return new Thread(r, "t" + num.getAndIncrement());
});
CountDownLatch latch = new CountDownLatch(10);
String[] all = new String[10];
Random r = new Random();
for (int j = 0; j < 10; j++) {
 int x = j;
 service.submit(() -> {
 for (int i = 0; i <= 100; i++) {
 try {
 Thread.sleep(r.nextInt(100));
 } catch (InterruptedException e) {
 }
 all[x] = Thread.currentThread().getName() + "(" + (i + "%") + ")";
 System.out.print("\r" + Arrays.toString(all));
 }
 latch.countDown();
 });
}
latch.await();
System.out.println("\n游戏开始...");
service.shutdown();
```

中间输出：

```java
[t0(52%), t1(47%), t2(51%), t3(40%), t4(49%), t5(44%), t6(49%), t7(52%), t8(46%), t9(46%)]

```

最后输出

```java
[t0(100%), t1(100%), t2(100%), t3(100%), t4(100%), t5(100%), t6(100%), t7(100%), t8(100%), 
t9(100%)] 
游戏开始...
```

#  **CyclicBarrier**

循环栅栏，用来进行线程协作，等待线程满足某个计数。构造时设置『计数个数』，每个线程执

行到某个需要“同步”的时刻调用 await() 方法进行等待，当等待的线程数满足『计数个数』时，继续执行

```java

@Slf4j(topic = "c.TestCyclicBarrier")
public class TestCyclicBarrier {
    public static void main(String[] args) {
        ExecutorService ser = Executors.newFixedThreadPool(2);
        CyclicBarrier cyclicBarrier = new CyclicBarrier(2, () -> {
            log.debug("task1 stask2 finish");
        });
        for (int i = 0; i < 3; i++) {
            ser.submit(()->{
                log.debug("task1 begin--");
                sleep(1);
                try {
                    cyclicBarrier.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (BrokenBarrierException e) {
                    e.printStackTrace();
                }

            });
            ser.submit(()->{
                log.debug("task2 begin--");
                sleep(2);
                try {
                    cyclicBarrier.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (BrokenBarrierException e) {
                    e.printStackTrace();
                }
            });
        }
        ser.shutdown();
    }
}
```

> **注意** CyclicBarrier 与 CountDownLatch 的主要区别在于 CyclicBarrier 是可以重用的CyclicBarrier可以被比喻为『人满发车』

# **线程安全集合类概述**

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20211201202838.png)



线程安全集合类可以分为三大类：

* 遗留的线程安全集合如 Hashtable ， Vector

* 使用 Collections 装饰的线程安全集合，如：

	* Collections.synchronizedCollection
	* Collections.synchronizedList
	* Collections.synchronizedMap
	* Collections.synchronizedSet
	* Collections.synchronizedNavigableMap
	* Collections.synchronizedNavigableSet 
	* Collections.synchronizedSortedMap
	* Collections.synchronizedSortedSet

* java.util.concurrent.*

重点介绍 java.util.concurrent.* 下的线程安全集合类，可以发现它们有规律，里面包含三类关键词：

Blocking、CopyOnWrite、Concurrent
* Blocking 大部分实现基于锁，并提供用来阻塞的方法
* CopyOnWrite 之类容器修改开销相对较重
* Concurrent 类型的容器
	* 内部很多操作使用 cas 优化，一般可以提供较高吞吐量
	* 弱一致性

		* 遍历时弱一致性，例如，当利用迭代器遍历时，如果容器发生修改，迭代器仍然可继续进行遍历，这时内容是旧的
		* 求大小弱一致性，size 操作未必是 100% 准确
		* 读取弱一致性

> 遍历时如果发生了修改，对于非安全容器来讲，使用 **fail-fast** 机制也就是让遍历立刻失败，抛出
> ConcurrentModifificationException，不再继续遍历

## **ConcurrentHashMap**  jdk 8

```java
// 默认为 0
// 当初始化时, 为 -1
// 当扩容时, 为 -(1 + 扩容线程数)
// 当初始化或扩容完成后，为 下一次的扩容的阈值大小
private transient volatile int sizeCtl;
// 整个 ConcurrentHashMap 就是一个 Node[]
static class Node<K,V> implements Map.Entry<K,V> {}
// hash 表
transient volatile Node<K,V>[] table;
// 扩容时的 新 hash 表
private transient volatile Node<K,V>[] nextTable;
// 扩容时如果某个 bin 迁移完毕, 用 ForwardingNode 作为旧 table bin 的头结点
static final class ForwardingNode<K,V> extends Node<K,V> {}
// 用在 compute 以及 computeIfAbsent 时, 用来占位, 计算完成后替换为普通 Node
static final class ReservationNode<K,V> extends Node<K,V> {}
// 作为 treebin 的头节点, 存储 root 和 first
static final class TreeBin<K,V> extends Node<K,V> {}
// 作为 treebin 的节点, 存储 parent, left, right
static final class TreeNode<K,V> extends Node<K,V> {}
```

### **重要方法**

```java
// 获取 Node[] 中第 i 个 Node
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i)
 
// cas 修改 Node[] 中第 i 个 Node 的值, c 为旧值, v 为新值
static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i, Node<K,V> c, Node<K,V> v)
 
// 直接修改 Node[] 中第 i 个 Node 的值, v 为新值
static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v)
```

**构造器分析**

可以看到实现了懒惰初始化，在构造方法中仅仅计算了 table 的大小，以后在第一次使用时才会真正创建

```java
public ConcurrentHashMap(int initialCapacity, float loadFactor, int concurrencyLevel) {
        if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
            throw new IllegalArgumentException();
        if (initialCapacity < concurrencyLevel) // Use at least as many bins
            initialCapacity = concurrencyLevel; // as estimated threads
        long size = (long) (1.0 + (long) initialCapacity / loadFactor);
        // tableSizeFor 仍然是保证计算的大小是 2^n, 即 16,32,64 ...
        int cap = (size >= (long) MAXIMUM_CAPACITY) ?
                MAXIMUM_CAPACITY : tableSizeFor((int) size);
        this.sizeCtl = cap;
}
```

**get** **流程**

```java
public V get(Object key) {
        Node<K, V>[] tab;
        Node<K, V> e, p;
        int n, eh;
        K ek;
        // spread 方法能确保返回结果是正数
        int h = spread(key.hashCode());
        if ((tab = table) != null && (n = tab.length) > 0 &&
                (e = tabAt(tab, (n - 1) & h)) != null) {
            // 如果头结点已经是要查找的 key
            if ((eh = e.hash) == h) {
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                    return e.val;
            }
            // hash 为负数表示该 bin 在扩容中或是 treebin, 这时调用 find 方法来查找
            else if (eh < 0)
                return (p = e.find(h, key)) != null ? p.val : null;
            // 正常遍历链表, 用 equals 比较
            while ((e = e.next) != null) {
                if (e.hash == h &&
                        ((ek = e.key) == key || (ek != null && key.equals(ek))))
                    return e.val;
            }
        }
        return null;
    }
```

**put** **流程**

以下数组简称（table），链表简称（bin）

```java
 public V put(K key, V value) {
        return putVal(key, value, false);
    }

    final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        // 其中 spread 方法会综合高位低位, 具有更好的 hash 性
        int hash = spread(key.hashCode());
        int binCount = 0;
        for (Node<K, V>[] tab = table; ; ) {
            // f 是链表头节点
            // fh 是链表头结点的 hash
            // i 是链表在 table 中的下标
            Node<K, V> f;
            int n, i, fh;
            // 要创建 table
            if (tab == null || (n = tab.length) == 0)
                // 初始化 table 使用了 cas, 无需 synchronized 创建成功, 进入下一轮循环
                tab = initTable();
                // 要创建链表头节点
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                // 添加链表头使用了 cas, 无需 synchronized
                if (casTabAt(tab, i, null,
                        new Node<K, V>(hash, key, value, null)))
                    break;
            }
            // 帮忙扩容
            else if ((fh = f.hash) == MOVED)
                // 帮忙之后, 进入下一轮循环
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
                // 锁住链表头节点
                synchronized (f) {
                    // 再次确认链表头节点没有被移动
                    if (tabAt(tab, i) == f) {
                        // 链表
                        if (fh >= 0) {
                            binCount = 1;
                            // 遍历链表
                            for (Node<K, V> e = f; ; ++binCount) {
                                K ek;
                                // 找到相同的 key
                                if (e.hash == hash &&
                                        ((ek = e.key) == key ||
                                                (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    // 更新
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K, V> pred = e;
                                // 已经是最后的节点了, 新增 Node, 追加至链表尾
                                if ((e = e.next) == null) {

                                    pred.next = new Node<K, V>(hash, key,
                                            value, null);
                                    break;
                                }
                            }
                        }
                        // 红黑树
                        else if (f instanceof TreeBin) {
                            Node<K, V> p;
                            binCount = 2;
                            // putTreeVal 会看 key 是否已经在树中, 是, 则返回对应的 TreeNode
                            if ((p = ((TreeBin<K, V>) f).putTreeVal(hash, key,
                                    value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                    // 释放链表头节点的锁
                }

                if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD)
                        // 如果链表长度 >= 树化阈值(8), 进行链表转为红黑树
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        // 增加 size 计数
        addCount(1L, binCount);
        return null;
    }

    private final Node<K, V>[] initTable() {
        Node<K, V>[] tab;
        int sc;
        while ((tab = table) == null || tab.length == 0) {
            if ((sc = sizeCtl) < 0)
                Thread.yield();
                // 尝试将 sizeCtl 设置为 -1（表示初始化 table）
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                // 获得锁, 创建 table, 这时其它线程会在 while() 循环中 yield 直至 table 创建
                try {
                    if ((tab = table) == null || tab.length == 0) {
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        Node<K, V>[] nt = (Node<K, V>[]) new Node<?, ?>[n];
                        table = tab = nt;
                        sc = n - (n >>> 2);
                    }
                } finally {
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }

    // check 是之前 binCount 的个数
    private final void addCount(long x, int check) {
        CounterCell[] as;
        long b, s;
        if (
            // 已经有了 counterCells, 向 cell 累加
                (as = counterCells) != null ||
                        // 还没有, 向 baseCount 累加
                        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)
        ) {
            CounterCell a;
            long v;
            int m;
            boolean uncontended = true;
            if (
                // 还没有 counterCells
                    as == null || (m = as.length - 1) < 0 ||
                            // 还没有 cell
                            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
                            // cell cas 增加计数失败
                            !(uncontended = U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))
            ) {
                // 创建累加单元数组和cell, 累加重试
                fullAddCount(x, uncontended);
                return;
            }
            if (check <= 1)
                return;
            // 获取元素个数
            s = sumCount();
        }
        if (check >= 0) {
            Node<K, V>[] tab, nt;
            int n, sc;
            while (s >= (long) (sc = sizeCtl) && (tab = table) != null &&
                    (n = tab.length) < MAXIMUM_CAPACITY) { 
                //为什么用while而不是if是因为可以判断新扩容的数组是不又超过了阈值。
                int rs = resizeStamp(n);
                if (sc < 0) {
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                            sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                            transferIndex <= 0)
                        break;
                    // newtable 已经创建了，帮忙扩容
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                        transfer(tab, nt);
                }
                // 需要扩容，这时 newtable 未创建
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                        (rs << RESIZE_STAMP_SHIFT) + 2)) transfer(tab, null);
                s = sumCount();
            }
        }
    }
 private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
        int n = tab.length, stride;  //stride 主要和CPU相关
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)  //每个线程处理桶的最小数目，可以看出核数越高步长越小，最小16个。
            stride = MIN_TRANSFER_STRIDE; // subdivide range
        if (nextTab == null) {
            try {
                @SuppressWarnings("unchecked")
                Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];  //扩容到2倍
                nextTab = nt;
            } catch (Throwable ex) {      // try to cope with OOME
                sizeCtl = Integer.MAX_VALUE;  //扩容保护
                return;
            }
            nextTable = nextTab;
            transferIndex = n;  //扩容总进度，>=transferIndex的桶都已分配出去。
        }
        int nextn = nextTab.length;
        //扩容时的特殊节点，标明此节点正在进行迁移，扩容期间的元素查找要调用其find()方法在nextTable中查找元素。
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
        //当前线程是否需要继续寻找下一个可处理的节点
        boolean advance = true;   //判断是否需要当前线程完成自己的步长（stride）后还需不需要进行数组的搬迁。
        boolean finishing = false; //所有桶是否都已迁移完成。
        for (int i = 0, bound = 0;;) {
            Node<K,V> f; int fh;
            //此循环的作用是确定当前线程要迁移的桶的范围或通过更新i的值确定当前范围内下一个要处理的节点。
            while (advance) {
                int nextIndex, nextBound;
                if (--i >= bound || finishing)  //每次循环都检查结束条件
                    advance = false;
                    //迁移总进度<=0，表示所有桶都已迁移完成。
                else if ((nextIndex = transferIndex) <= 0) {
                    i = -1;
                    advance = false;
                }
                else if (U.compareAndSwapInt
                        (this, TRANSFERINDEX, nextIndex,
                                nextBound = (nextIndex > stride ?
                                        nextIndex - stride : 0))) {  //transferIndex减去已分配出去的桶。
                    //确定当前线程每次分配的待迁移桶的范围为[bound, nextIndex)
                    bound = nextBound;
                    i = nextIndex - 1;
                    advance = false;
                }
            }
            //当前线程自己的活已经做完或所有线程的活都已做完，第二与第三个条件应该是下面让"i = n"后，再次进入循环时要做的边界检查。
            if (i < 0 || i >= n || i + n >= nextn) {
                int sc;
                if (finishing) {  //所有线程已干完活，最后才走这里。
                    nextTable = null;
                    table = nextTab;  //替换新table
                    sizeCtl = (n << 1) - (n >>> 1); //调sizeCtl为新容量0.75倍。
                    return;
                }
                //当前线程已结束扩容，sizeCtl-1表示参与扩容线程数-1。
                if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                    //还记得addCount()处给sizeCtl赋的初值吗？相等时说明没有线程在参与扩容了，置finishing=advance=true，为保险让i=n再检查一次。
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        return;
                    finishing = advance = true;
                    i = n; // recheck before commit
                }
            }
            else if ((f = tabAt(tab, i)) == null)
                advance = casTabAt(tab, i, null, fwd);  //如果i处是ForwardingNode表示第i个桶已经有线程在负责迁移了。
            else if ((fh = f.hash) == MOVED)
                advance = true; // already processed
            else {
                synchronized (f) {  //桶内元素迁移需要加锁。
                    if (tabAt(tab, i) == f) {
                        Node<K,V> ln, hn;
                        //链表的处理
                        if (fh >= 0) {  //>=0表示是链表结点
                            //由于n是2的幂次方（所有二进制位中只有一个1)，如n=16(0001 0000)，第4位为1，那么hash&n后的值第4位只能为0或1。所以可以根据hash&n的结果将所有结点分为两部分。
                            //如果runbit等于1就表示当前节点，要放在新表的高位，0就是地位
                            int runBit = fh & n;
                            Node<K,V> lastRun = f;  
                            //找出最后一段完整的fh&n不变的链表，这样最后这一段链表就不用重新创建新结点了。
                            for (Node<K,V> p = f.next; p != null; p = p.next) {
                                int b = p.hash & n;
                                //求出最后一段是在新表中连续的链表的头节点。
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
                                //runbit等于1时反在新表的高位
                                hn = lastRun;
                                ln = null;
                            }
                            //lastRun之前的结点因为fh&n不确定，所以全部需要重新迁移。
                            for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                int ph = p.hash; K pk = p.key; V pv = p.val;
                                if ((ph & n) == 0)  //接在低位
                                    ln = new Node<K,V>(ph, pk, pv, ln);
                                else //接在高位
                                    hn = new Node<K,V>(ph, pk, pv, hn);
                            }
                            //低位链表放在i处
                            setTabAt(nextTab, i, ln);
                            //高位链表放在i+n处
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);  //在原table中设置ForwardingNode节点以提示该桶扩容完成。
                            advance = true;
                        }
                        //红黑树的处理
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
                            //如果拆分后的树的节点数量已经少于6个就需要重新转化为链表
                            ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                                    (hc != 0) ? new TreeBin<K,V>(lo) : t;
                            hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                                    (lc != 0) ? new TreeBin<K,V>(hi) : t;
                            //CAS存储在nextTable的i位置上
                            setTabAt(nextTab, i, ln);
                            //CAS存储在nextTable的i+n位置上
                            setTabAt(nextTab, i + n, hn);
                            //CAS在原table的i处设置forwordingNode节点，表示这个这个节点已经处理完毕
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                    }
                }
            }
        }
    }


    
```

**size** **计算流程**

size 计算实际发生在 put，remove 改变集合元素的操作之中

* 没有竞争发生，向 baseCount 累加计数
* 有竞争发生，新建 counterCells，向其中的一个 cell 累加计数

	* counterCells 初始有两个 cell
* 如果计数竞争比较激烈，会创建新的 cell 来累加计数

```java
public int size() {
        long n = sumCount();
        return ((n < 0L) ? 0 :
                (n > (long) Integer.MAX_VALUE) ? Integer.MAX_VALUE :
                        (int) n);
    }

    final long sumCount() {
        CounterCell[] as = counterCells;
        CounterCell a;
        // 将 baseCount 计数与所有 cell 计数累加
        long sum = baseCount;
        if (as != null) {
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null)
                    sum += a.value;
            }
        }
        return sum;
    }

```

**Java 8** 数组（Node） +（ 链表 Node | 红黑树 TreeNode ） 以下数组简称（table），链表简称（bin）

* 初始化，使用 cas 来保证并发安全，懒惰初始化 table
* 树化，当 table.length < 64 时，先尝试扩容，超过 64 时，并且 bin.length > 8 时，会将链表树化，树化过程会用 synchronized 锁住链表头
* put，如果该 bin 尚未创建，只需要使用 cas 创建 bin；如果已经有了，锁住链表头进行后续 put 操作，元素添加至 bin 的尾部
* get，无锁操作仅需要保证可见性，扩容过程中 get 操作拿到的是 ForwardingNode 它会让 get 操作在新table 进行搜索
* 扩容，扩容时以 bin 为单位进行，需要对 bin 进行 synchronized，但这时妙的是其它竞争线程也不是无事可做，它们会帮助把其它 bin 进行扩容，扩容时平均只有 1/6 的节点会把复制到新 table 中
* size，元素个数保存在 baseCount 中，并发时的个数变动保存在 CounterCell[] 当中。最后统计数量时累加即

## **ConcurrentHashMap**   jdk7

它维护了一个 segment 数组，每个 segment 对应一把锁

优点：如果多个线程访问不同的 segment，实际是没有冲突的，这与 jdk8 中是类似的

缺点：Segments 数组默认大小为16，这个容量初始化指定后就不能改变了，并且不是懒惰初始化

**构造器分析**

```java
public ConcurrentHashMap(int initialCapacity, float loadFactor, int concurrencyLevel) {
        if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
        if (concurrencyLevel > MAX_SEGMENTS)
        concurrencyLevel = MAX_SEGMENTS;
        // ssize 必须是 2^n, 即 2, 4, 8, 16 ... 表示了 segments 数组的大小
        int sshift = 0;
        int ssize = 1;
        while (ssize < concurrencyLevel) {
        ++sshift;
        ssize <<= 1;
        }
        // segmentShift 默认是 32 - 4 = 28
        this.segmentShift = 32 - sshift;
        // segmentMask 默认是 15 即 0000 0000 0000 1111
        this.segmentMask = ssize - 1;
        if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
        int c = initialCapacity / ssize;
        if (c * ssize < initialCapacity)
        ++c;
        int cap = MIN_SEGMENT_TABLE_CAPACITY;
        while (cap < c)
        cap <<= 1;
        // 创建 segments and segments[0]
        Segment<K,V> s0 =
        new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
        (HashEntry<K,V>[])new HashEntry[cap]);
        Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
        UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segments[0]
        this.segments = ss; }
```

构造完成，如下图所示

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20211202205745.png)

可以看到 ConcurrentHashMap 没有实现懒惰初始化，空间占用不友好

其中 this.segmentShift 和 this.segmentMask 的作用是决定将 key 的 hash 结果匹配到哪个 segment

例如，根据某一 hash 值求 segment 位置，先将高位向低位移动 this.segmentShift 位

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20211202205804.png)

**put** **流程**

```java
public V put(K key, V value) {
        Segment<K,V> s;
        if (value == null)
            throw new NullPointerException();
        int hash = hash(key);
        // 计算出 segment 下标
        int j = (hash >>> segmentShift) & segmentMask;

        // 获得 segment 对象, 判断是否为 null, 是则创建该 segment
        if ((s = (Segment<K,V>)UNSAFE.getObject
                (segments, (j << SSHIFT) + SBASE)) == null) {
            // 这时不能确定是否真的为 null, 因为其它线程也发现该 segment 为 null,
            // 因此在 ensureSegment 里用 cas 方式保证该 segment 安全性
            s = ensureSegment(j);
        }
        // 进入 segment 的put 流程
        return s.put(key, hash, value, false);
    }
```

segment 继承了可重入锁（ReentrantLock），它的 put 方法为

```java
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
        // 尝试加锁
        HashEntry<K, V> node = tryLock() ? null :
                // 如果不成功, 进入 scanAndLockForPut 流程
                // 如果是多核 cpu 最多 tryLock 64 次, 进入 lock 流程
                // 在尝试期间, 还可以顺便看该节点在链表中有没有, 如果没有顺便创建出来
                scanAndLockForPut(key, hash, value);

        // 执行到这里 segment 已经被成功加锁, 可以安全执行
        V oldValue;
        try {
            HashEntry<K, V>[] tab = table;
            int index = (tab.length - 1) & hash;
            HashEntry<K, V> first = entryAt(tab, index);
            for (HashEntry<K, V> e = first; ; ) {
                if (e != null) {
                    // 更新
                    K k;
                    if ((k = e.key) == key ||
                            (e.hash == hash && key.equals(k))) {
                        oldValue = e.value;
                        if (!onlyIfAbsent) {
                            e.value = value;
                            ++modCount;
                        }
                        break;
                    }
                    e = e.next;
                } else {
                    // 新增
                    // 1) 之前等待锁时, node 已经被创建, next 指向链表头
                    if (node != null)
                        node.setNext(first);
                    else
                        // 2) 创建新 node
                        node = new HashEntry<K, V>(hash, key, value, first);
                    int c = count + 1;
                    // 3) 扩容
                    if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                        rehash(node);
                    else
                        // 将 node 作为链表头
                        setEntryAt(tab, index, node);
                    ++modCount;
                    count = c;
                    oldValue = null;
                    break;
                }
            }
        } finally {
            unlock();
        }
        return oldValue;
    }
```

**rehash** **流程**

发生在 put 中，因为此时已经获得了锁，因此 rehash 时不需要考虑线程安全

```java
private void rehash(HashEntry<K, V> node) {
        HashEntry<K, V>[] oldTable = table;
        int oldCapacity = oldTable.length;
        int newCapacity = oldCapacity << 1;
        threshold = (int) (newCapacity * loadFactor);
        HashEntry<K, V>[] newTable =
                (HashEntry<K, V>[]) new HashEntry[newCapacity];
        int sizeMask = newCapacity - 1;
        for (int i = 0; i < oldCapacity; i++) {
            HashEntry<K, V> e = oldTable[i];
            if (e != null) {
                HashEntry<K, V> next = e.next;
                int idx = e.hash & sizeMask;
                if (next == null) // Single node on list
                    newTable[idx] = e;
                else { // Reuse consecutive sequence at same slot
                    HashEntry<K, V> lastRun = e;
                    int lastIdx = idx;
                    // 过一遍链表, 尽可能把 rehash 后 idx 不变的节点重用
                    for (HashEntry<K, V> last = next;
                         last != null;
                         last = last.next) {
                        int k = last.hash & sizeMask;
                        if (k != lastIdx) {
                            lastIdx = k;
                            lastRun = last;
                        }
                    }
                    newTable[lastIdx] = lastRun;
                    // 剩余节点需要新建
                    for (HashEntry<K, V> p = e; p != lastRun; p = p.next) {
                        V v = p.value;
                        int h = p.hash;
                        int k = h & sizeMask;
                        HashEntry<K, V> n = newTable[k];
                        newTable[k] = new HashEntry<K, V>(h, p.key, v, n);
                    }
                }
            }
        }
        // 扩容完成, 才加入新的节点
        int nodeIndex = node.hash & sizeMask; // add the new node
        node.setNext(newTable[nodeIndex]);
        newTable[nodeIndex] = node;

        // 替换为新的 HashEntry table
        table = newTable;
    }
```

**get** **流程**

get 时并未加锁，用了 UNSAFE 方法保证了可见性，扩容过程中，get 先发生就从旧表取内容，get 后发生就从新

表取内容

```java
public V get(Object key) {
        Segment<K, V> s; // manually integrate access methods to reduce overhead
        HashEntry<K, V>[] tab;
        int h = hash(key);
        // u 为 segment 对象在数组中的偏移量
        long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
        // s 即为 segment
        if ((s = (Segment<K, V>) UNSAFE.getObjectVolatile(segments, u)) != null &&
                (tab = s.table) != null) {
            for (HashEntry<K, V> e = (HashEntry<K, V>) UNSAFE.getObjectVolatile
                    (tab, ((long) (((tab.length - 1) & h)) << TSHIFT) + TBASE);
                 e != null; e = e.next) {
                K k;
                if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                    return e.value;
            }
        }
        return null;
    }
```

**size** **计算流程**

* 计算元素个数前，先不加锁计算两次，如果前后两次结果如一样，认为个数正确返回

* 如果不一样，进行重试，重试次数超过 3，将所有 segment 锁住，重新计算个数返回

```java
public int size() {
        // Try a few times to get accurate count. On failure due to
        // continuous async changes in table, resort to locking.
        final Segment<K, V>[] segments = this.segments;
        int size;
        boolean overflow; // true if size overflows 32 bits
        long sum; // sum of modCounts
        long last = 0L; // previous sum
        int retries = -1; // first iteration isn't retry
        try {
            for (; ; ) {
                if (retries++ == RETRIES_BEFORE_LOCK) {
                    // 超过重试次数, 需要创建所有 segment 并加锁
                    for (int j = 0; j < segments.length; ++j)
                        ensureSegment(j).lock(); // force creation
                }
                sum = 0L;
                size = 0;
                overflow = false;
                for (int j = 0; j < segments.length; ++j) {
                    Segment<K, V> seg = segmentAt(segments, j);
                    if (seg != null) {
                        sum += seg.modCount;
                        int c = seg.count;
                        if (c < 0 || (size += c) < 0)
                            overflow = true;
                    }
                }
                if (sum == last)
                    break;
                last = sum;
            }
        } finally {
            if (retries > RETRIES_BEFORE_LOCK) {
                for (int j = 0; j < segments.length; ++j)
                    segmentAt(segments, j).unlock();
            }
        }
        return overflow ? Integer.MAX_VALUE : size;
    }
```

