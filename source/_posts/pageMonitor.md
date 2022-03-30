---
title: Monitor和join原理和wait/notifty原理和park等
date: 2021-11-20 20:11:04
tags: Java并发
categories: java基础
---
## java对象头
普通对象
![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20211120204330.png)

数组对象：
![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20211120204350.png)

其中 Mark Word 结构为

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20211120204434.png)

64位虚拟机：
![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20211120204502.png)

## 原理之 Monitor(锁)
Monitor 被翻译为监视器或管程  
每个 Java 对象都可以关联一个 Monitor 对象，如果使用 synchronized 给对象上锁（重量级）之后，该对象头的Mark Word 中就被设置指向 Monitor 对象的指针  
![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20211120203639.png)
* 刚开始 Monitor 中 Owner 为 null
* 当 Thread-2 执行 synchronized(obj) 就会将 Monitor 的所有者 Owner 置为 Thread-2，Monitor中只能有一个 Owner  
* 在 Thread-2 上锁的过程中，如果 Thread-3，Thread-4，Thread-5 也来执行 synchronized(obj)，就会进入EntryList BLOCKED  
* Thread-2 执行完同步代码块的内容，然后唤醒 EntryList 中等待的线程来竞争锁，竞争的时是非公平的  
* 图中 WaitSet 中的 Thread-0，Thread-1 是之前获得过锁，但条件不满足进入 WAITING 状态的线程，后面讲wait-notify 时会分析  

>注意：
>>synchronized 必须是进入同一个对象的 monitor 才有上述的效果
>>不加 synchronized 的对象不会关联监视器，不遵从以上规则

```java
static final Object lock = new Object();
static int counter = 0;
public static void main(String[] args) {
 synchronized (lock) {
 counter++;
 }
}
```

字节码：
```
public static void main(java.lang.String[]);
 descriptor: ([Ljava/lang/String;)V
 flags: ACC_PUBLIC, ACC_STATIC
Code:
 stack=2, locals=3, args_size=1
 0: getstatic #2 // <- lock引用 （synchronized开始）
 3: dup
 4: astore_1 // lock引用 -> slot 1
 5: monitorenter // 将 lock对象 MarkWord 置为 Monitor 指针
 6: getstatic #3 // <- i
 9: iconst_1 // 准备常数 1
 10: iadd // +1
 11: putstatic #3 // -> i
 14: aload_1 // <- lock引用
 15: monitorexit // 将 lock对象 MarkWord 重置, 唤醒 EntryList
 16: goto 24
 19: astore_2 // e -> slot 2 
 20: aload_1 // <- lock引用
 21: monitorexit // 将 lock对象 MarkWord 重置, 唤醒 EntryList
 22: aload_2 // <- slot 2 (e)
 23: athrow // throw e
 24: return
 Exception table:
 from to target type
 6 16 19 any
 19 22 19 any
 LineNumberTable:
 line 8: 0
 line 9: 6
 line 10: 14
 line 11: 24
 LocalVariableTable:
 Start Length Slot Name Signature
 0 25 0 args [Ljava/lang/String;
 StackMapTable: number_of_entries = 2
 frame_type = 255 /* full_frame */
 offset_delta = 19
 locals = [ class "[Ljava/lang/String;", class java/lang/Object ]
 stack = [ class java/lang/Throwable ]
 frame_type = 250 /* chop */
 offset_delta = 4
```

方法级别的 synchronized 不会在字节码指令中有所体现

## 锁的升级：

**小故事**
故事角色
老王 - JVM
小南 - 线程
小女 - 线程
房间 - 对象
房间门上 - 防盗锁 - Monitor
房间门上 - 小南书包 - 轻量级锁
房间门上 - 刻上小南大名 - 偏向锁
批量重刻名 - 一个类的偏向锁撤销到达 20 阈值
不能刻名字 - 批量撤销该类对象的偏向锁，设置该类不可偏向

小南要使用房间保证计算不被其它人干扰（原子性），最初，他用的是防盗锁，当上下文切换时，锁住门。这样，
即使他离开了，别人也进不了门，他的工作就是安全的。
但是，很多情况下没人跟他来竞争房间的使用权。小女是要用房间，但使用的时间上是错开的，小南白天用，小女
晚上用。每次上锁太麻烦了，有没有更简单的办法呢？
小南和小女商量了一下，约定不锁门了，而是谁用房间，谁把自己的书包挂在门口，但他们的书包样式都一样，因
此每次进门前得翻翻书包，看课本是谁的，如果是自己的，那么就可以进门，这样省的上锁解锁了。万一书包不是
自己的，那么就在门外等，并通知对方下次用锁门的方式。
后来，小女回老家了，很长一段时间都不会用这个房间。小南每次还是挂书包，翻书包，虽然比锁门省事了，但仍
然觉得麻烦。
于是，小南干脆在门上刻上了自己的名字：【小南专属房间，其它人勿用】，下次来用房间时，只要名字还在，那
么说明没人打扰，还是可以安全地使用房间。如果这期间有其它人要用这个房间，那么由使用者将小南刻的名字擦
掉，升级为挂书包的方式。
同学们都放假回老家了，小南就膨胀了，在 20 个房间刻上了自己的名字，想进哪个进哪个。后来他自己放假回老
家了，这时小女回来了（她也要用这些房间），结果就是得一个个地擦掉小南刻的名字，升级为挂书包的方式。老
王觉得这成本有点高，提出了一种批量重刻名的方法，他让小女不用挂书包了，可以直接在门上刻上自己的名字
后来，刻名的现象越来越频繁，老王受不了了：算了，这些房间都不能刻名了，只能挂书包

###  轻量级锁（没有竞争）：
1. 轻量级锁
轻量级锁的使用场景：如果一个对象虽然有多线程要加锁，但加锁的时间是错开的（也就是没有竞争），那么可以
使用轻量级锁来优化。
轻量级锁对使用者是透明的，即语法仍然是 synchronized
假设有两个方法同步块，利用同一个对象加锁
```java

static final Object obj = new Object();
public static void method1() {
 synchronized( obj ) {
 // 同步块 A
 method2();
 }
}
public static void method2() {
 synchronized( obj ) {
 // 同步块 B
 }
}
```
* 创建锁记录（Lock Record）对象，每个线程的栈帧都会包含一个锁记录的结构，内部可以存储锁定对象的
Mark Word
![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20211120205907.png)

* 让锁记录中 Object reference 指向锁对象，并尝试用 cas 替换 Object 的 Mark Word，将 Mark Word 的值存入锁记录
![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20211120205951.png)

* 如果 cas 替换成功，对象头中存储了 锁记录地址和状态 00 ，表示由该线程给对象加锁，这时图示如下
 ![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20211120210018.png)

* 如果 cas 失败，有两种情况
	* 如果是其它线程已经持有了该 Object 的轻量级锁，这时表明有竞争，进入锁膨胀过程
	*如果是自己执行了 synchronized 锁重入，那么再添加一条 Lock Record 作为重入的计数
![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20211120210115.png)

* 当退出 synchronized 代码块（解锁时）如果有取值为 null 的锁记录，表示有重入，这时重置锁记录，表示重入计数减一
 ![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20211120210142.png)

* 当退出 synchronized 代码块（解锁时）锁记录的值不为 null，这时使用 cas 将 Mark Word 的值恢复给对象头
	* 成功，则解锁成功
	* 失败，说明轻量级锁进行了锁膨胀或已经升级为重量级锁，进入重量级锁解锁流程
	

###  锁膨胀：
如果在尝试加轻量级锁的过程中，CAS 操作无法成功，这时一种情况就是有其它线程为此对象加上了轻量级锁（有
竞争），这时需要进行锁膨胀，将轻量级锁变为重量级锁。

```java
static Object obj = new Object();
public static void method1() {
 synchronized( obj ) {
 // 同步块
 }
}
```
* 当 Thread-1 进行轻量级加锁时，Thread-0 已经对该对象加了轻量级锁
![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20211120211443.png)

* 这时 Thread-1 加轻量级锁失败，进入锁膨胀流程
	* 即为 Object 对象申请 Monitor 锁，让 Object 指向重量级锁地址
	* 然后自己进入 Monitor 的 EntryList BLOCKED
![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20211120211503.png)
* 当 Thread-0 退出同步块解锁时，使用 cas 将 Mark Word 的值恢复给对象头，失败。这时会进入重量级解锁
流程，即按照 Monitor 地址找到 Monitor 对象，设置 Owner 为 null，唤醒 EntryList 中 BLOCKED 线程


###  自旋优化

重量级锁竞争的时候，还可以使用自旋来进行优化，如果当前线程自旋成功（即这时候持锁线程已经退出了同步
块，释放了锁），这时当前线程就可以避免阻塞。
![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20211121143559.png)

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20211121143620.png)

* 自旋会占用 CPU 时间，单核 CPU 自旋就是浪费，多核 CPU 自旋才能发挥优势。
* 在 Java 6 之后自旋锁是自适应的，比如对象刚刚的一次自旋操作成功过，那么认为这次自旋成功的可能性会高，就多自旋几次；反之，就少自旋甚至不自旋，总之，比较智能。
* Java 7 之后不能控制是否开启自旋功能

###  偏向锁（没有竞争升级为轻量级锁（00），有的话就升级为重量级锁（10））
**偏向锁是默认是延迟的**

轻量级锁在没有竞争时（就自己这个线程），每次重入仍然需要执行 CAS 操作。
Java 6 中引入了偏向锁来做进一步优化：只有第一次使用 CAS 将线程 ID 设置到对象的 Mark Word 头，之后发现
这个线程 ID 是自己的就表示没有竞争，不用重新 CAS。以后只要不发生竞争，这个对象就归该线程所有
```java

static final Object obj = new Object();
public static void m1() {
 synchronized( obj ) {
 // 同步块 A
 m2();
 }
}
public static void m2() {
 synchronized( obj ) {
 // 同步块 B
 m3();
 }
}
public static void m3() {
 synchronized( obj ) {
// 同步块 C
 }
}
```

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20211121150109.png)
![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20211121150056.png)
回忆对象头：
![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20211120204502.png)


一个对象创建时：  
* 如果开启了偏向锁（默认开启），那么对象创建后，markword 值为 0x05 即最后 3 位为 101，这时它的
thread、epoch、age 都为 0  
* 偏向锁是默认是延迟的，不会在程序启动时立即生效，如果想避免延迟，可以加 VM 参数-XX:BiasedLockingStartupDelay=0 来禁用延迟  
* 如果没有开启偏向锁，那么对象创建后，markword 值为 0x01 即最后 3 位为 001，这时它的 hashcode、age 都为 0，第一次用到 hashcode 时才会赋值

测试：使用第三方jar包
```
Maven:

<dependency>
     <groupId>org.openjdk.jol</groupId>
     <artifactId>jol-core</artifactId>
     <version>0.9</version>
</dependency>
```

代码：
```java
public class TestBasied {

    public static void main(String[] args) throws InterruptedException {

        Thread.sleep(4000);
        log.debug("{}", ClassLayout.parseInstance(new Dog()).toPrintable());


    }
}
class Dog{

}
```

执行结果：
![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20211121151345.png)


使用XX:BiasedLockingStartupDelay=0就不需要睡眠4秒了。
 
* **偏向锁的撤销**  
	* **调用对象的hashCode方法会撤销，偏向锁，因为偏向锁状态Mark中没有保存hashcode。**
	* **撤销 - 其它线程使用对象**
		案例：
```java
public class TestBasied2 {
    public static void main(String[] args) throws InterruptedException {
        Object obj= new Object();
        new Thread(()->{
            synchronized (obj){
                
            }
        },"t1").start();
        log.debug("{}", ClassLayout.parseInstance(obj).toPrintable());
        new Thread(()->{
            log.debug("{}", ClassLayout.parseInstance(obj).toPrintable());
            synchronized (obj){
                log.debug("{}", ClassLayout.parseInstance(obj).toPrintable());
            }
            log.debug("{}", ClassLayout.parseInstance(obj).toPrintable());
        },"t2").start();
    }
}
```
	* **撤销 - 调用 wait/notify**


###  批量重偏向
如果对象虽然被多个线程访问，但没有竞争，这时偏向了线程 T1 的对象仍有机会重新偏向 T2，重偏向会重置对象
的 Thread ID
当撤销偏向锁阈值超过 20 次后，jvm 会这样觉得，我是不是偏向错了呢，于是会在给这些对象加锁时重新偏向至
加锁线程

```java
private static void test3() throws InterruptedException {
        Vector<Dog> list = new Vector<>();
        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 30; i++) {
                Dog d = new Dog();
                list.add(d);
                synchronized (d) {
                    log.debug(i + "\t" + ClassLayout.parseInstance(d).toPrintable());
                }
            }
            synchronized (list) {
                list.notify();
            }
        }, "t1");
        t1.start();

        Thread t2 = new Thread(() -> {
            synchronized (list) {
                try {
                    list.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            log.debug("===============> ");
            for (int i = 0; i < 30; i++) {
                Dog d = list.get(i);
                log.debug(i + "\t" + ClassLayout.parseInstance(d).toPrintable());
                synchronized (d) {
                    log.debug(i + "\t" + ClassLayout.parseInstance(d).toPrintable());
                }
                log.debug(i + "\t" + ClassLayout.parseInstance(d).toPrintable());
            }
        }, "t2");
        t2.start();
    }

```

###  批量撤销
当撤销偏向锁阈值超过 40 次后，jvm 会这样觉得，自己确实偏向错了，根本就不该偏向。于是整个类的所有对象
都会变为不可偏向的，新建的对象也是不可偏向的

```java
static Thread t1,t2,t3;
    private static void test4() throws InterruptedException {
        Vector<Dog> list = new Vector<>();
        int loopNumber = 39;
        t1 = new Thread(() -> {
            for (int i = 0; i < loopNumber; i++) {
                Dog d = new Dog();
                list.add(d);
                synchronized (d) {
                    log.debug(i + "\t" + ClassLayout.parseInstance(d).toPrintable());
                }
            }
            LockSupport.unpark(t2);
        }, "t1");
        t1.start();
        t2 = new Thread(() -> {
            LockSupport.park();
            log.debug("===============> ");
            for (int i = 0; i < loopNumber; i++) {
                Dog d = list.get(i);
                log.debug(i + "\t" + ClassLayout.parseInstance(d).toPrintable());
                synchronized (d) {
                    log.debug(i + "\t" + ClassLayout.parseInstance(d).toPrintable());
                }
                log.debug(i + "\t" + ClassLayout.parseInstance(d).toPrintable());
            }
            LockSupport.unpark(t3);
        }, "t2");
        t2.start();
        t3 = new Thread(() -> {
            LockSupport.park();
            log.debug("===============> ");
            for (int i = 0; i < loopNumber; i++) {
                Dog d = list.get(i);
                log.debug(i + "\t" + ClassLayout.parseInstance(d).toPrintable());
                synchronized (d) {
                    log.debug(i + "\t" + ClassLayout.parseInstance(d).toPrintable());
                }
                log.debug(i + "\t" + ClassLayout.parseInstance(d).toPrintable());
            }
        }, "t3");
        t3.start();
        t3.join();
        log.debug(ClassLayout.parseInstance(new Dog()).toPrintable());
        //可以看到超过了阈值 然后把这个类创建的所有对象都是不可偏向的

    }

```
###  锁消除

和逃逸分析很像。
如果对象没有逃离方法，而你却用它做了锁对象，那么即时编译器，就会消除这个锁。



## wait/notify原理
![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20211121183400.png)
* Owner 线程发现条件不满足，调用 wait 方法，即可进入 WaitSet 变为 WAITING 状态  
* BLOCKED 和 WAITING 的线程都处于阻塞状态，不占用 CPU 时间片  
* BLOCKED 线程会在 Owner 线程释放锁时唤醒   
* WAITING 线程会在 Owner 线程调用 notify 或 notifyAll 时唤醒，但唤醒后并不意味者立刻获得锁，仍需进入EntryList 重新竞争  



**API介绍：**

* obj.wait() 让进入 object 监视器的线程到 waitSet 等待
* obj.notify() 在 object 上正在 waitSet 等待的线程中挑一个唤醒
* obj.notifyAll() 让 object 上正在 waitSet 等待的线程全部唤醒

它们都是线程之间进行协作的手段，都属于 Object 对象的方法。必须获得此对象的锁，才能调用这几个方法
```java
 final static Object obj=new Object();

    public static void main(String[] args) throws InterruptedException {
        new Thread(()->{
            log.debug("执行");
            synchronized (obj){
                try {
                    obj.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                log.debug("其他代码执行");
            }
        }
        ,"t1").start();
        new Thread(()->{
            synchronized (obj){
                log.debug("执行");

                try {
                    obj.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                log.debug("其他代码执行");

            }
        },"t2").start();
        Thread.sleep(1000);
        log.debug("唤醒obj上其他线程");
        synchronized (obj){
            obj.notifyAll();
        }
    }

```

执行结果：
```
19:25:44.019 c.Test18 [t2] - 执行
19:25:44.019 c.Test18 [t1] - 执行
19:25:45.018 c.Test18 [main] - 唤醒obj上其他线程
19:25:45.018 c.Test18 [t1] - 其他代码执行
19:25:45.018 c.Test18 [t2] - 其他代码执行
```


##  park&UnPark

基本使用
它们是 LockSupport 类中的方法
```java
// 暂停当前线程
LockSupport.park(); 
// 恢复某个线程的运行
LockSupport.unpark(暂停线程对象)

```
**与 Object 的 wait & notify 相比**
* wait，notify 和 notifyAll 必须配合 Object Monitor 一起使用，而 park，unpark 不必
* park & unpark 是以线程为单位来【阻塞】和【唤醒】线程，而 notify 只能随机唤醒一个等待线程，notifyAll是唤醒所有等待线程，就不那么【精确】
* park & unpark 可以先 unpark，而 wait & notify 不能先 notify



## **原理：park&UnPark**

每个线程都有自己的一个 Parker 对象，由三部分组成 _counter ， _cond 和 _mutex 打个比喻
*线程就像一个旅人，Parker 就像他随身携带的背包，条件变量就好比背包中的帐篷。_counter 就好比背包中
的备用干粮（0 为耗尽，1 为充足）
* 调用 park 就是要看需不需要停下来歇息
	* 如果备用干粮耗尽，那么钻进帐篷歇息
	* 如果备用干粮充足，那么不需停留，继续前进
* 调用 unpark，就好比令干粮充足
	* 如果这时线程还在帐篷，就唤醒让他继续前进
	* 如果这时线程还在运行，那么下次他调用 park 时，仅是消耗掉备用干粮，不需停留继续前进
* 因为背包空间有限，多次调用 unpark 仅会补充一份备用干粮

1. 当前线程调用 Unsafe.park() 方法
2. 检查 _counter ，本情况为 0，这时，获得 _mutex 互斥锁
3. 线程进入 _cond 条件变量阻塞
4. 设置 _counter = 0
![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20211122233003.png)



1. 调用 Unsafe.unpark(Thread_0) 方法，设置 _counter 为 1
2. 唤醒 _cond 条件变量中的 Thread_0
3. Thread_0 恢复运行
4. 设置 _counter 为 0
![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20211122233052.png)
