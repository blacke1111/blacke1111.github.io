---
title: ReentrantReadWriteLock
date: 2021-11-29 23:05:07
tags: Java并发
categories: java基础
---

# 读写锁

##  ReentrantReadWriteLock

当读操作远远高于写操作时，这时候使用 读写锁 让 读-读 可以并发，提高性能。 类似于数据库中的 select ...from ... lock in share mode

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

使用读写锁实现一个简单的按需加载缓存

在读锁的时候要使用二级检查



```java
class GenericCachedDao<T> {
    // HashMap 作为缓存非线程安全, 需要保护
    HashMap<SqlPair, T> map = new HashMap<>();

    ReentrantReadWriteLock lock = new ReentrantReadWriteLock();
    GenericDao genericDao = new GenericDao();
    public int update(String sql, Object... params) {
        SqlPair key = new SqlPair(sql, params);
        // 加写锁, 防止其它线程对缓存读取和更改
        lock.writeLock().lock();
        try {
            int rows = genericDao.update(sql, params);
            map.clear();
            return rows;
        } finally {
            lock.writeLock().unlock();
        }
    }
    public T queryOne(Class<T> beanClass, String sql, Object... params) {
        SqlPair key = new SqlPair(sql, params);
        // 加读锁, 防止其它线程对缓存更改
        lock.readLock().lock();
        try {
            T value = map.get(key);
            if (value != null) {
                return value;
            }
        } finally {
            lock.readLock().unlock();
        }
        // 加写锁, 防止其它线程对缓存读取和更改
        lock.writeLock().lock();
        try {
            // get 方法上面部分是可能多个线程进来的, 可能已经向缓存填充了数据
            // 为防止重复查询数据库, 再次验证
            T value = map.get(key);
            if (value == null) {
                // 如果没有, 查询数据库
                value = genericDao.queryOne(beanClass, sql, params);
                map.put(key, value);
            }
            return value;
        } finally {
            lock.writeLock().unlock();
        }
    }
    // 作为 key 保证其是不可变的
    class SqlPair {
        private String sql;
        private Object[] params;
        public SqlPair(String sql, Object[] params) {
            this.sql = sql;
            this.params = params;
        }
        @Override
        public boolean equals(Object o) {
            if (this == o) {
                return true;
            }
            if (o == null || getClass() != o.getClass()) {
                return false;
            }
            SqlPair sqlPair = (SqlPair) o;
            return sql.equals(sqlPair.sql) &&
                    Arrays.equals(params, sqlPair.params);
        }
        @Override
        public int hashCode() {
            int result = Objects.hash(sql);
            result = 31 * result + Arrays.hashCode(params);
            return result;
        }
    }
}
```

> **注意**
> 
> * 以上实现体现的是读写锁的应用，保证缓存和数据库的一致性，但有下面的问题没有考虑
> 
> 	* 适合读多写少，如果写操作比较频繁，以上实现性能低
> 	* 没有考虑缓存容量
> 	* 没有考虑缓存过期
> 	* 只适合单机
> 	* 并发性还是低，目前只会用一把锁 
> 	* 更新方法太过简单粗暴，清空了所有 key（考虑按类型分区或重新设计 key）
> 
> * 乐观锁实现：用 CAS 去更新

# 读写锁原理

## **1.** **图解流程**

读写锁用的是同一个 Sycn 同步器，因此等待队列、state 等也是同一个

### **t1 w.lock, t2 r.lock**

1） t1 成功上锁，流程与 ReentrantLock 加锁相比没有特殊之处，不同是写锁状态占了 state 的低 16 位，而读锁

使用的是 state 的高 16 位 

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20211130205756.png)

2）t2 执行 r.lock，这时进入读锁的 sync.acquireShared(1) 流程，首先会进入 tryAcquireShared 流程。如果有写锁占据，那么 tryAcquireShared 返回 -1 表示失败
> tryAcquireShared 返回值表示
> * -1 表示失败
> * 0 表示成功，但后继节点不会继续唤醒
> * 正数表示成功，而且数值是还有几个后继节点需要唤醒，读写锁返回 1* 

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20211130205900.png)

3）这时会进入 sync.doAcquireShared(1) 流程，首先也是调用 addWaiter 添加节点，不同之处在于节点被设置为Node.SHARED 模式而非 Node.EXCLUSIVE 模式，注意此时 t2 仍处于活跃状态

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20211130205922.png)

4）t2 会看看自己的节点是不是老二，如果是，还会再次调用 tryAcquireShared(1) 来尝试获取锁

5）如果没有成功，在 doAcquireShared 内 for (;;) 循环一次，把前驱节点的 waitStatus 改为 -1，再 for (;;) 循环一次尝试 tryAcquireShared(1) 如果还不成功，那么在 parkAndCheckInterrupt() 处 park

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20211130205942.png)

### **t3 r.lock，t4 w.lock**

这种状态下，假设又有 t3 加读锁和 t4 加写锁，这期间 t1 仍然持有锁，就变成了下面的样子

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20211130210024.png)

### **t1 w.unlock**

这时会走到写锁的 sync.release(1) 流程，调用 sync.tryRelease(1) 成功，变成下面的样子

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20211130210044.png)

接下来执行唤醒流程 sync.unparkSuccessor，即让老二恢复运行，这时 t2 在 doAcquireShared 内parkAndCheckInterrupt() 处恢复运行

这回再来一次 for (;;) 执行 tryAcquireShared 成功则让读锁计数加一

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20211130210107.png)

这时 t2 已经恢复运行，接下来 t2 调用 setHeadAndPropagate(node, 1)，它原本所在节点被置为头节点

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20211130210126.png)

事情还没完，在 setHeadAndPropagate 方法内还会检查下一个节点是否是 shared，如果是则调用

doReleaseShared() 将 head 的状态从 -1 改为 0 并唤醒老二，这时 t3 在 doAcquireShared 内

parkAndCheckInterrupt() 处恢复运行

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20211130210143.png)

这回再来一次 for (;;) 执行 tryAcquireShared 成功则让读锁计数加一

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20211130210157.png)

这时 t3 已经恢复运行，接下来 t3 调用 setHeadAndPropagate(node, 1)，它原本所在节点被置为头节点

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20211130210224.png)

下一个节点不是 shared 了，因此不会继续唤醒 t4 所在节点

### **t2 r.unlock,  t3 r.unlock**

t2 进入 sync.releaseShared(1) 中，调用 tryReleaseShared(1) 让计数减一，但由于计数还不为零

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20211130210247.png)

t3 进入 sync.releaseShared(1) 中，调用 tryReleaseShared(1) 让计数减一，这回计数为零了，进入

doReleaseShared() 将头节点从 -1 改为 0 并唤醒老二，即

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20211130210303.png)

之后 t4 在 acquireQueued 中 parkAndCheckInterrupt 处恢复运行，再次 for (;;) 这次自己是老二，并且没有其他

竞争，tryAcquire(1) 成功，修改头结点，流程结束

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20211130210320.png)

**读图时一定切记配合源码**
