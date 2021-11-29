---
title: ReentrantReadWriteLock
date: 2021-11-29 23:05:07
tags: Java并发
categories: java基础
---

# 读写锁

##  ReentrantReadWriteLock

当读操作远远高于写操作时，这时候使用 读写锁 让 读-读 可以并发，提高性能。 类似于数据库中的 select ...

from ... lock in share mode

提供一个 数据容器类 内部分别使用读锁保护数据的 read() 方法，写锁保护数据的 write() 方法



```java
class DataContainer {
 private Object data;
 private ReentrantReadWriteLock rw = new ReentrantReadWriteLock();
 private ReentrantReadWriteLock.ReadLock r = rw.readLock();
 private ReentrantReadWriteLock.WriteLock w = rw.writeLock();
 public Object read() {
 log.debug("获取读锁...");
 r.lock();
 try {
 log.debug("读取");
 sleep(1);
 return data;
 } finally {
 log.debug("释放读锁...");
     r.unlock();
 }
 }
 public void write() {
 log.debug("获取写锁...");
 w.lock();
 try {
 log.debug("写入");
 sleep(1);
 } finally {
 log.debug("释放写锁...");
 w.unlock();
 }
 }
}
```

测试 读锁-读锁 可以并发

```java
DataContainer dataContainer = new DataContainer();
new Thread(() -> {
 dataContainer.read();
}, "t1").start();
new Thread(() -> {
 dataContainer.read();
}, "t2").start();
```

输出结果，从这里可以看到 Thread-0 锁定期间，Thread-1 的读操作不受影响

```java
14:05:14.341 c.DataContainer [t2] - 获取读锁... 
14:05:14.341 c.DataContainer [t1] - 获取读锁... 
14:05:14.345 c.DataContainer [t1] - 读取
14:05:14.345 c.DataContainer [t2] - 读取
14:05:15.365 c.DataContainer [t2] - 释放读锁... 
14:05:15.386 c.DataContainer [t1] - 释放读锁.
```

测试 读锁-写锁 相互阻塞

```java
DataContainer dataContainer = new DataContainer();
new Thread(() -> {
 dataContainer.read();
}, "t1").start();
Thread.sleep(100);
new Thread(() -> {
 dataContainer.write();
}, "t2").start();
```

输出结果

```java
14:04:21.838 c.DataContainer [t1] - 获取读锁... 
14:04:21.838 c.DataContainer [t2] - 获取写锁... 
14:04:21.841 c.DataContainer [t2] - 写入
14:04:22.843 c.DataContainer [t2] - 释放写锁... 
14:04:22.843 c.DataContainer [t1] - 读取
14:04:23.843 c.DataContainer [t1] - 释放读锁... 
```

写锁-写锁 也是相互阻塞的，这里就不测试了



**注意事项**

* 读锁不支持条件变量* 
* 重入时升级不支持：即持有读锁的情况下去获取写锁，会导致获取写锁永久等待

```java
r.lock();
try {
 // ...
 w.lock();
 try {
 // ...
 } finally{
 w.unlock();
 }
} finally{
 r.unlock();
}
```

* 重入时降级支持：即持有写锁的情况下去获取读锁

```java
class CachedData {
 Object data;
 // 是否有效，如果失效，需要重新计算 data
 volatile boolean cacheValid;
 final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
 void processCachedData() {
 rwl.readLock().lock();
 if (!cacheValid) {
 // 获取写锁前必须释放读锁
 rwl.readLock().unlock();
 rwl.writeLock().lock();
 try {
 // 判断是否有其它线程已经获取了写锁、更新了缓存, 避免重复更新
 if (!cacheValid) {
 data = ...
 cacheValid = true;
 }
 // 降级为读锁, 释放写锁, 这样能够让其它线程读取缓存
 rwl.readLock().lock();
 } finally {
     rwl.writeLock().unlock();
 }
 }
 // 自己用完数据, 释放读锁 
 try {
 use(data);
 } finally {
 rwl.readLock().unlock();
 }
 }
}
```

# **缓存**

更新时，是先清缓存还是先更新数据库

先清缓存

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20211129232507.png)

先更新数据库

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20211129232530.png)

补充一种情况，假设查询线程 A 查询数据时恰好缓存数据由于时间到期失效，或是第一次查询

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20211129232546.png)

这种情况的出现几率非常小，见 facebook 论文

#  **读写锁实现一致性缓存**

