---
title: LinkedBlockingQueue和ConcurrentLinkedQueue
date: 2021-12-02 21:36:19
tags: Java并发
categories: java基础
---

# LinkedBlockingQueue

## 1.基本的入队出队

```java
public class LinkedBlockingQueue<E> extends AbstractQueue<E>
 implements BlockingQueue<E>, java.io.Serializable {
 static class Node<E> {
     E item;
 /**
 * 下列三种情况之一
 * - 真正的后继节点
 * - 自己, 发生在出队时
 * - null, 表示是没有后继节点, 是最后了
 */
 Node<E> next;
 Node(E x) { item = x; }
 }
}
```

初始化链表 last = head = new Node<E>(null); Dummy 节点用来占位，item 为 null

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20211202213830.png)

当一个节点入队 last = last.next = node;

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20211202213854.png)

再来一个节点入队 last = last.next = node;

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20211202213954.png)

```java
Node<E> h = head;
Node<E> first = h.next; h.next = h; // help GC
head = first; E x = first.item;
first.item = null;
return x;
```

**h = head**

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20211202214029.png)

**first = h.next**

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20211202214050.png)

**h.next = h**

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20211202214108.png)

**head = first**

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20211202214125.png)

```java
E x = first.item;
first.item = null;
return x;
```

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20211202214212.png)



## **2.** **加锁分析**

==高明之处==在于用了两把锁和 dummy 节点

* 用一把锁，同一时刻，最多只允许有一个线程（生产者或消费者，二选一）执行
* 用两把锁，同一时刻，可以允许两个线程同时（一个生产者与一个消费者）执行
	* 消费者与消费者线程仍然串行
	* 生产者与生产者线程仍然串行

线程安全分析

* 当节点总数大于 2 时（包括 dummy 节点），putLock 保证的是 last 节点的线程安全，takeLock 保证的是head 节点的线程安全。两把锁保证了入队和出队没有竞争
* 当节点总数等于 2 时（即一个 dummy 节点，一个正常节点）这时候，仍然是两把锁锁两个对象，不会竞争
* 当节点总数等于 1 时（就一个 dummy 节点）这时 take 线程会被 notEmpty 条件阻塞，有竞争，会阻塞

```java
// 用于 put(阻塞) offer(非阻塞)
private final ReentrantLock putLock = new ReentrantLock();
// 用户 take(阻塞) poll(非阻塞)
private final ReentrantLock takeLock = new ReentrantLock();
```

put 操作



```java
public void put(E e) throws InterruptedException {
    if (e == null) throw new NullPointerException();
 int c = -1;
 Node<E> node = new Node<E>(e);
 final ReentrantLock putLock = this.putLock;
 // count 用来维护元素计数
 final AtomicInteger count = this.count;
 putLock.lockInterruptibly();
 try {
 // 满了等待
 while (count.get() == capacity) {
 // 倒过来读就好: 等待 notFull
 notFull.await();
 }
 // 有空位, 入队且计数加一
 enqueue(node);
 c = count.getAndIncrement(); 
 // 除了自己 put 以外, 队列还有空位, 由自己叫醒其他 put 线程
 if (c + 1 < capacity)
 notFull.signal();
 } finally {
 putLock.unlock();
 }
 // 如果队列中有一个元素, 叫醒 take 线程
 if (c == 0)
 // 这里调用的是 notEmpty.signal() 而不是 notEmpty.signalAll() 是为了减少竞争
 signalNotEmpty();
}
```

take 操作

```java
public E take() throws InterruptedException {
 E x;
 int c = -1;
 final AtomicInteger count = this.count;
 final ReentrantLock takeLock = this.takeLock;
 takeLock.lockInterruptibly();
 try {
 while (count.get() == 0) {
 notEmpty.await();
 }
 x = dequeue();
 c = count.getAndDecrement();
 if (c > 1)
 notEmpty.signal();
 } finally {
 takeLock.unlock();
 }
 // 如果队列中只有一个空位时, 叫醒 put 线程
 // 如果有多个线程进行出队, 第一个线程满足 c == capacity, 但后续线程 c < capacity
 if (c == capacity)
 // 这里调用的是 notFull.signal() 而不是 notFull.signalAll() 是为了减少竞争
 signalNotFull()
     return x; }
```



## 3. **性能比较**

主要列举 LinkedBlockingQueue 与 ArrayBlockingQueue 的性能比较
* Linked 支持有界，Array 强制有界
* Linked 实现是链表，Array 实现是数组
* Linked 是懒惰的，而 Array 需要提前初始化 Node 数组
* Linked 每次入队会生成新 Node，而 Array 的 Node 是提前创建好的
* Linked 两把锁，Array 一把锁





#  ConcurrentLinkedQueue

ConcurrentLinkedQueue 的设计与 LinkedBlockingQueue 非常像，也是

* 两把【锁】，同一时刻，可以允许两个线程同时（一个生产者与一个消费者）执行
* dummy 节点的引入让两把【锁】将来锁住的是不同对象，避免竞争
* 只是这【锁】使用了 cas 来实现

事实上，ConcurrentLinkedQueue 应用还是非常广泛的
例如之前讲的 Tomcat 的 Connector 结构时，Acceptor 作为生产者向 Poller 消费者传递事件信息时，正是采用了
ConcurrentLinkedQueue 将 SocketChannel 给 Poller 使用



tomcat:

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20211202214542.png)

# CopyOnWriteArraySet 

是它的马甲 底层实现采用了 写入时拷贝 的思想，增删改操作会将底层数组拷贝一份，更

改操作在新数组上执行，这时不影响其它线程的**并发读**，**读写分离**。 以新增为例：

```java
public boolean add(E e) {
 synchronized (lock) {
 // 获取旧的数组
 Object[] es = getArray();
 int len = es.length;
 // 拷贝新的数组（这里是比较耗时的操作，但不影响其它读线程）
 es = Arrays.copyOf(es, len + 1);
 // 添加新元素
 es[len] = e;
 // 替换旧的数组
 setArray(es);
 return true;
 }
}
```

> 这里的源码版本是 Java 11，在 Java 1.8 中使用的是可重入锁而不是 synchronized

其它读操作并未加锁，例如：`

```java
public void forEach(Consumer<? super E> action) {
 Objects.requireNonNull(action);
 for (Object x : getArray()) {
 @SuppressWarnings("unchecked") E e = (E) x;
 action.accept(e);
 }
}
```

适合『读多写少』的应用场景

**get** **弱一致性**

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20211202215411.png)

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20211202215426.png)

> 不容易测试，但问题确实存在

**迭代器弱一致性**

```java
CopyOnWriteArrayList<Integer> list = new CopyOnWriteArrayList<>();
list.add(1);
list.add(2);
list.add(3);
Iterator<Integer> iter = list.iterator();
new Thread(() -> {
 list.remove(0);
 System.out.println(list);
}).start();
sleep1s();
while (iter.hasNext()) {
 System.out.println(iter.next());
}
```

不要觉得弱一致性就不好

* 数据库的 MVCC 都是弱一致性的表现

* 并发高和一致性是矛盾的，需要权衡
