---
title: Java线程的使用
date: 2021-11-16 19:01:21
tags: Java并发
categories: java基础
---


## 方法一:直接使用

```java
// 创建线程对象
Thread t = new Thread() {
 	public void run() {
 	// 要执行的任务
 	}
};
// 启动线程
t.start();
```


## 方法二：使用 Runnable 配合 Thread
```java
Runnable runnable = new Runnable() {
 public void run(){
 // 要执行的任务
 }
};
// 创建线程对象
Thread t = new Thread( runnable );
// 启动线程
t.start();

```


Java 8 以后可以使用 lambda 精简代码:  
注意：@FunctionalInterface注解的接口可以使用lambda表达式

```java
// 创建任务对象
Runnable task2 = () -> log.debug("hello");
// 参数1 是任务对象; 参数2 是线程名字，推荐
Thread t2 = new Thread(task2, "t2");
t2.start();
```

小结
* 方法1 是把线程和任务合并在了一起，方法2 是把线程和任务分开了
* 用 Runnable 更容易与线程池等高级 API 配合
* 用 Runnable 让任务类脱离了 Thread 继承体系，更灵活

## 方法三，FutureTask 配合 Thread

FutureTask 能够接收 Callable 类型的参数，用来处理有返回结果的情况
```java

@Slf4j(topic = "c.Test3")
public class Test3 {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        FutureTask futureTask = new FutureTask<Integer>(new Callable<Integer>() {
            @Override
            public Integer call() throws Exception {
                log.debug("call running");
                Thread.sleep(1000);
                return 100;
            }
        });

        Thread thread = new Thread(futureTask,"t1");
        thread.start();

        System.out.println(futureTask.get());
    }
}
```


## 查看进程线程的方法


**windows**
任务管理器可以查看进程和线程数，也可以用来杀死进程  
`tasklist` 查看进程  
`taskkill` 杀死进程

**linux**  
`ps -fe` 查看所有进程  
`ps -fT -p <PID>` 查看某个进程（PID）的所有线程  
`kill` 杀死进程  
`top` 按大写 H 切换是否显示线程  
`top -H -p <PID> `查看某个进程（PID）的所有线程  
**Java**  
`jps `命令查看所有 Java 进程  
`jstack` <PID> 查看某个 Java 进程（PID）的所有线程状态  
`jconsole` 来查看某个 Java 进程中线程的运行情况（图形界面）  

**jconsole 远程监控配置**
需要以如下方式运行你的 java 类

    
    java -Djava.rmi.server.hostname=192.168.32.3 -Dcom.sun.management.jmxremote -
    Dcom.sun.management.jmxremote.port=12345 -Dcom.sun.management.jmxremote.ssl=false -
	Dcom.sun.management.jmxremote.authenticate=false java类名 

记住要关闭防火墙 **systemctl stop firewalld**

* 修改 /etc/hosts 文件将 127.0.0.1 映射至主机名
* 如果要认证访问，还需要做如下步骤
* 复制 jmxremote.password 文件
* 修改 jmxremote.password 和 jmxremote.access 文件的权限为 600 即文件所有者可读写
* 连接时填入 controlRole（用户名），R&D（密码）