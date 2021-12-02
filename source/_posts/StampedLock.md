---
title: StampedLock
date: 2021-11-30 21:46:55
tags: Java并发
categories: java基础
---

#  **StampedLock**

该类自 JDK 8 加入，是为了进一步优化读性能，它的特点是在使用读锁、写锁时都必须配合【戳】使用加解读锁

```java
long stamp = lock.readLock();
lock.unlockRead(stamp);
```

加解写锁

```java
long stamp = lock.writeLock();
lock.unlockWrite(stamp);
```

乐观读，StampedLock 支持 tryOptimisticRead() 方法（乐观读），读取完毕后需要做一次 戳校验 如果校验通过，表示这期间确实没有写操作，数据可以安全使用，如果校验没通过，需要重新获取读锁，保证数据安全。



```java
long stamp = lock.tryOptimisticRead();
// 验戳
if(!lock.validate(stamp)){
 // 锁升级
}
```

提供一个 数据容器类 内部分别使用读锁保护数据的 read() 方法，写锁保护数据的 write() 方法

```java
class DataContainerStamped {
 private int data;
    private final StampedLock lock = new StampedLock();
 public DataContainerStamped(int data) {
 this.data = data;
 }
 public int read(int readTime) {
 long stamp = lock.tryOptimisticRead();
 log.debug("optimistic read locking...{}", stamp);
 sleep(readTime);
 if (lock.validate(stamp)) {
 log.debug("read finish...{}, data:{}", stamp, data);
 return data;
 }
 // 锁升级 - 读锁
 log.debug("updating to read lock... {}", stamp);
 try {
 stamp = lock.readLock();
 log.debug("read lock {}", stamp);
 sleep(readTime);
 log.debug("read finish...{}, data:{}", stamp, data);
 return data;
 } finally {
 log.debug("read unlock {}", stamp);
 lock.unlockRead(stamp);
 }
 }
 public void write(int newData) {
 long stamp = lock.writeLock();
 log.debug("write lock {}", stamp);
 try {
 sleep(2);
 this.data = newData;
 } finally {
 log.debug("write unlock {}", stamp);
 lock.unlockWrite(stamp);
 }
 }
}

```



测试 读-读 可以优化

```java
public static void main(String[] args) {
 DataContainerStamped dataContainer = new DataContainerStamped(1);
 new Thread(() -> {
 dataContainer.read(1);
 }, "t1").start();
 sleep(0.5);
 new Thread(() -> {
 dataContainer.read(0);
 }, "t2").start();
}
```



输出结果，可以看到实际没有加读锁

```java
15:58:50.217 c.DataContainerStamped [t1] - optimistic read locking...256 
15:58:50.717 c.DataContainerStamped [t2] - optimistic read locking...256 
15:58:50.717 c.DataContainerStamped [t2] - read finish...256, data:1 
15:58:51.220 c.DataContainerStamped [t1] - read finish...256, data:1
```

测试 读-写 时优化读补加读锁

```java
public static void main(String[] args) {
 DataContainerStamped dataContainer = new DataContainerStamped(1);
 new Thread(() -> {
 dataContainer.read(1);
 }, "t1").start();
 sleep(0.5);
 new Thread(() -> {
 dataContainer.write(100);
 }, "t2").start();
}
```

输出结果

```java
15:57:00.219 c.DataContainerStamped [t1] - optimistic read locking...256 
15:57:00.717 c.DataContainerStamped [t2] - write lock 384 
15:57:01.225 c.DataContainerStamped [t1] - updating to read lock... 256 
15:57:02.719 c.DataContainerStamped [t2] - write unlock 384 
15:57:02.719 c.DataContainerStamped [t1] - read lock 513 
15:57:03.719 c.DataContainerStamped [t1] - read finish...513, data:1000 
15:57:03.719 c.DataContainerStamped [t1] - read unlock 513
```



> **注意**
> StampedLock 不支持条件变量
> StampedLock 不支持可重入

# **Semaphore**

信号量，用来限制能同时访问共享资源的线程上限。

```java
public static void main(String[] args) {
 // 1. 创建 semaphore 对象
 Semaphore semaphore = new Semaphore(3);
 // 2. 10个线程同时运行
 for (int i = 0; i < 10; i++) {
 new Thread(() -> {
 // 3. 获取许可
 try {
 semaphore.acquire();
 } catch (InterruptedException e) {
 e.printStackTrace();
 }
 try {
 log.debug("running...");
 sleep(1);
 log.debug("end...");
 } finally {
 // 4. 释放许可
 semaphore.release();
 }
 }).start();
 }
 }
```

输出：

```java
07:35:15.485 c.TestSemaphore [Thread-2] - running... 
07:35:15.485 c.TestSemaphore [Thread-1] - running... 
07:35:15.485 c.TestSemaphore [Thread-0] - running... 
07:35:16.490 c.TestSemaphore [Thread-2] - end... 
07:35:16.490 c.TestSemaphore [Thread-0] - end... 
07:35:16.490 c.TestSemaphore [Thread-1] - end... 
07:35:16.490 c.TestSemaphore [Thread-3] - running... 
07:35:16.490 c.TestSemaphore [Thread-5] - running... 
07:35:16.490 c.TestSemaphore [Thread-4] - running... 
07:35:17.490 c.TestSemaphore [Thread-5] - end... 
07:35:17.490 c.TestSemaphore [Thread-4] - end... 
07:35:17.490 c.TestSemaphore [Thread-3] - end... 
07:35:17.490 c.TestSemaphore [Thread-6] - running... 
07:35:17.490 c.TestSemaphore [Thread-7] - running... 
07:35:17.490 c.TestSemaphore [Thread-9] - running... 
07:35:18.491 c.TestSemaphore [Thread-6] - end... 
07:35:18.491 c.TestSemaphore [Thread-7] - end... 
07:35:18.491 c.TestSemaphore [Thread-9] - end... 
07:35:18.491 c.TestSemaphore [Thread-8] - running... 
07:35:19.492 c.TestSemaphore [Thread-8] - end...
```



# Semaphore 原理

**加锁解锁流程**

Semaphore 有点像一个停车场，permits 就好像停车位数量，当线程获得了 permits 就像是获得了停车位，然后停车场显示空余车位减一刚开始，permits（state）为 3，这时 5 个线程来获取资源

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20211130213950.png)

假设其中 Thread-1，Thread-2，Thread-4 cas 竞争成功，而 Thread-0 和 Thread-3 竞争失败，进入 AQS 队列park 阻塞

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20211130214016.png)

这时 Thread-4 释放了 permits，状态如下

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20211130214043.png)

接下来 Thread-0 竞争成功，permits 再次设置为 0，设置自己为 head 节点，断开原来的 head 节点，unpark 接下来的 Thread-3 节点，但由于 permits 是 0，因此 Thread-3 在尝试不成功后再次进入 park 状态

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20211130222257.png)
