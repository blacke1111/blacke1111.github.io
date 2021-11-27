---
title: Unsafe
date: 2021-11-26 19:12:54
tags: Java并发
categories: java基础
---
之前学习的无锁用于实现线程安全的原子整数和原子引用等底层实现都是使用了Unsafe，这可看出他的重要性，他可以直接操作内存，看他名字不是说他是线程不安全的，而是让程序员一般不要去使用它。

虽然它的设计模式是单例的但是他的实例对象需要通过反射获取，不能直接调用Unsafe的getUnsafe（）方法，具体原因看源码：
![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20211126201322.png)

这里面有一个方法VM.isSystemDomainLoader(var0.getClassLoader())他的主要目的就是判断当前类加载器是不是引导类加载器（Bootstarp Classloader）如果不是就会直接抛出一个异常。
在我们java里获得引导类加载器是获取不到的，因为他是c/c++编写的。
![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20211126201546.png)

下面图片来自我学习jVM虚拟机的截图，笔记做在印象笔记上，没有传到博客园，视频链接：[尚硅谷宋红康JVM全套教程（详解java虚拟机）](https://www.bilibili.com/video/BV1PJ411n7xZ?from=search&seid=6601940611042531810&spm_id_from=333.337.0.0)
![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20211126201801.png)
![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20211126201820.png)
![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20211126201833.png)




# 获取Unsafe对象
反射：
```java
	Field unsafe1 = Unsafe.class.getDeclaredField("theUnsafe");
	//设置私有属性可见
    unsafe1.setAccessible(true);
    unsafe= (Unsafe) unsafe1.get(null);

```
# cas实现之AtomicInteger（一部分）

```
/**
 * @author zhy
 * @date 2021/11/26 19:29
 */
public class MyAtomicInteger {
    private static final Unsafe unsafe;
    private  final static Long valueOffest;
    private  volatile  int value;

    public MyAtomicInteger(int initialValue) {
        this.value = initialValue;
    }

    static {
        try {
            Field unsafe1 = Unsafe.class.getDeclaredField("theUnsafe");
            unsafe1.setAccessible(true);
             unsafe= (Unsafe) unsafe1.get(null);
            valueOffest=unsafe.objectFieldOffset(MyAtomicInteger.class.getDeclaredField("value"));
        } catch (NoSuchFieldException | IllegalAccessException e) {
            e.printStackTrace();
            throw  new RuntimeException();
        }
    }

    public int getValue(){
        return  value;
    }

    public  int getAndUpdate(IntUnaryOperator amount){
        int pre ,next;
        while (true){
            pre=getValue();
            next=amount.applyAsInt(pre);
            if (compareAndSet(pre,next)){
                break;
            }
        }
        return pre;
    }

    public boolean compareAndSet(int expect,int next){
        return unsafe.compareAndSwapInt(this,valueOffest,expect,next);
    }

}
```
