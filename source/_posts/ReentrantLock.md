---
title: ReentrantLock
date: 2021-11-23 18:23:09
tags: Java并发
categories: java基础
---
# 基本语法

相对于 synchronized 它具备如下特点
* 可中断
* 可以设置超时时间
* 可以设置为公平锁
* 支持多个条件变量
与 synchronized 一样，都支持可重入
基本语法
```java
// 获取锁
reentrantLock.lock();
try {
 // 临界区
} finally {
 // 释放锁
 reentrantLock.unlock();
}

```

## 可重入:
可重入是指同一个线程如果首次获得了这把锁，那么因为它是这把锁的拥有者，因此有权利再次获取这把锁
如果是不可重入锁，那么第二次获得锁时，自己也会被锁挡住
```java
static ReentrantLock lock = new ReentrantLock();
public static void main(String[] args) {
 method1();
}
public static void method1() {
 lock.lock();
 try {
 log.debug("execute method1");
 method2();
 } finally {
 lock.unlock();
 }
}
public static void method2() {
 lock.lock();
 try {
 log.debug("execute method2");
 method3();
 } finally {
 lock.unlock();
 }
}
public static void method3() {
 lock.lock();
 try {
 log.debug("execute method3");
 } finally {
 lock.unlock();
 }
}

```
输出：
```java
17:59:11.862 [main] c.TestReentrant - execute method1 
17:59:11.865 [main] c.TestReentrant - execute method2 
17:59:11.865 [main] c.TestReentrant - execute method3
```
## 可打断
```java
@Slf4j(topic = "c.Test22")
public class Test22 {
    private static ReentrantLock reent=new ReentrantLock();
    public static void main(String[] args) {
      Thread t1=  new Thread(()->{
            try {
                log.debug("尝试获得锁");
                reent.lockInterruptibly();
            } catch (InterruptedException e) {
                log.debug("获得锁失败");
                e.printStackTrace();
                return;
            }
            try {

            }finally {
                reent.unlock();
            }

        },"t1");
        reent.lock();
        t1.start();

        Sleeper.sleep(1);

        t1.interrupt();
    }
}


```
输出：
```java
18:44:33.186 c.Test22 [t1] - 尝试获得锁
18:44:34.185 c.Test22 [t1] - 获得锁失败
java.lang.InterruptedException
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireInterruptibly(AbstractQueuedSynchronizer.java:898)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireInterruptibly(AbstractQueuedSynchronizer.java:1222)
	at java.util.concurrent.locks.ReentrantLock.lockInterruptibly(ReentrantLock.java:335)
	at cn.itcast.mytest.Test22.lambda$main$0(Test22.java:19)
	at java.lang.Thread.run(Thread.java:748)
```
注意如果是不可中断模式，那么即使使用了 interrupt 也不会让等待中断

## 尝试获得锁：
* tryLock()立刻获得锁，如果失败就返回false
* tryLock(long timeout, TimeUnit unit)等待timeout时间去获得锁，如果在这个时间段内获得了锁就返回true，反之false。

```java
@Slf4j(topic = "c.Test23")
public class Test23 {
    private  static ReentrantLock lock=new ReentrantLock();
    public static void main(String[] args) {
        Thread thread = new Thread(() -> {
            log.debug("尝试获得锁");
            try {
                while (!lock.tryLock(3, TimeUnit.SECONDS)) {
                    log.debug("尝试获得锁失败");
                    break;
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            try {
                log.debug("获得了锁");
            }finally {
                lock.unlock();
            }
        }, "t1");
        log.debug("main县城获得了锁");
        lock.lock();
        thread.start();
        Sleeper.sleep(2);
        lock.unlock();
    }
}

```

## 解决哲学家就餐问题：

```java
public class EatingExperts {
    public static void main(String[] args) {
        Chopsticks c1 = new Chopsticks("c1");
        Chopsticks c2 = new Chopsticks("c2");
        Chopsticks c3 = new Chopsticks("c3");
        Chopsticks c4 = new Chopsticks("c4");
        Chopsticks c5 = new Chopsticks("c5");
        new Export("亚里士多德", c1, c2).start();
        new Export("马克思", c2, c3).start();
        new Export("鲁迅", c3, c4).start();
        new Export("毛泽东", c4, c5).start();
        new Export("马云", c5, c1).start();
    }
}

@Slf4j(topic = "c.Export")
class Export extends  Thread{
    private  String name;
    private  Chopsticks left;
    private  Chopsticks right;

    public Export(String name, Chopsticks left, Chopsticks right) {
        super(name);
        this.name=name;
        this.left = left;
        this.right = right;
    }

    @Override
    public void run() {
        while (true)
            eating();
    }

    public void eating(){
        if (left.tryLock()){
            try{
                if (right.tryLock()){
                 try {
                     log.debug("{}正在就餐。。。。",name);
                 }finally {
                     right.unlock();
                 }
                }
            }finally {
                left.unlock();
            }
        }
    }
}
class Chopsticks extends  ReentrantLock{
   private String name;

    public Chopsticks() {
    }

    public Chopsticks(String name) {
        this.name = name;
    }

    @Override
    public String toString() {
        return "chopsticks{" +
                "name='" + name + '\'' +
                '}';
    }
}

```


## 公平锁

ReentrantLock 默认是不公平的

```java
public boolean tryLock(long timeout, TimeUnit unit)
            throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(timeout));
    }

```
可以看到ReentrantLock的构造方法可以让由于等待锁而进入阻塞队列的线程重新获得锁的时候争抢时公平的。


## 条件变量
synchronized 中也有条件变量，就是我们讲原理时那个 waitSet 休息室，当条件不满足时进入 waitSet 等待
ReentrantLock 的条件变量比 synchronized 强大之处在于，它是支持多个条件变量的，这就好比
* synchronized 是那些不满足条件的线程都在一间休息室等消息
* 而 ReentrantLock 支持多间休息室，有专门等烟的休息室、专门等早餐的休息室、唤醒时也是按休息室来唤
醒
使用要点：
* await 前需要获得锁
* await 执行后，会释放锁，进入 conditionObject 等待
* await 的线程被唤醒（或打断、或超时）取重新竞争 lock 锁
* 竞争 lock 锁成功后，从 await 后继续执行

```java
public class Test24 {
    static ReentrantLock lock = new ReentrantLock();
    static Condition waitCigaretteQueue = lock.newCondition();
    static Condition waitbreakfastQueue = lock.newCondition();
    static volatile boolean hasCigrette = false;
    static volatile boolean hasBreakfast = false;
    public static void main(String[] args) {
        new Thread(() -> {
            try {
                lock.lock();
                while (!hasCigrette) {
                    try {
                        waitCigaretteQueue.await();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                log.debug("等到了它的烟");
            } finally {
                lock.unlock();
            }
        }).start();
        new Thread(() -> {
            try {
                lock.lock();
                while (!hasBreakfast) {
                    try {
                        waitbreakfastQueue.await();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                log.debug("等到了它的早餐");
            } finally {
                lock.unlock();
            }
        }).start();
        sleep(1);
        sendBreakfast();
        sleep(1);
        sendCigarette();
    }
    private static void sendCigarette() {
        lock.lock();
        try {
            log.debug("送烟来了");
            hasCigrette = true;
            waitCigaretteQueue.signal();
        } finally {
            lock.unlock();
        }
    }
    private static void sendBreakfast() {
        lock.lock();
        try {
            log.debug("送早餐来了");
            hasBreakfast = true;
            waitbreakfastQueue.signal();
        } finally {
            lock.unlock();
        }
    }
}
```

