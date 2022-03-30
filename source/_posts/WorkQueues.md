---
title: WorkQueues
date: 2022-01-06 17:39:13
categories: RabbitMQ
---



## 轮询机制：

启动两个消费者，一个生产者， 

这两个消费者轮询的消费生产者生产的消息。

生产者：

```java
public class Task01 {
    private static final String QUEUE_NAME="hello";
    public static void main(String[] args) throws  Exception{
        Channel channel = RabbitMqUtils.getChannel();
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        System.out.println("输入消息：");
        Scanner scanner=new Scanner(System.in);
        while (scanner.hasNext()) {
            String message = scanner.next();
            channel.basicPublish("", QUEUE_NAME, null, message.getBytes());
            System.out.println("消息发送完成");
        }
    }
}
```

消费者： 我这里是直接使用了idea启动2个Woker01类

```java
public class Woker01 {
    private static final String QUEUE_NAME="hello";
    public static void main(String[] args) throws  Exception{
        Channel channel = RabbitMqUtils.getChannel();
        channel.basicConsume(QUEUE_NAME, true,
                //consumerTag消费的一个标记 delivery传递给消费者的传递内容
                (consumerTag, delivery) -> {
                    System.out.println("消费者a1:"+consumerTag);
                    String string = new String(delivery.getBody());
                    System.out.println("接受的消息：" + string);
                    long deliveryTag = delivery.getEnvelope().getDeliveryTag();
                    System.out.println("编号：" + deliveryTag);

                },
                consumerTag -> {
                    System.out.println(consumerTag + "消费者：取消消费消息");
                });
    }
}
```

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220106174234.png)

勾选上面的箭头所指选项，就可以启动2个类并行。

## 消息应答



**A**.Channel.basicAck(用于肯定确认)

​	RabbitMQ 已知道该消息并且成功的处理消息，可以将其丢弃了

**B**.Channel.basicNack(用于否定确认) 

**C**.Channel.basicReject(用于否定确认) 

​		与 Channel.basicNack 相比少一个参数

​		不处理该消息了直接拒绝，可以将其丢弃了

.Channel.basicAck 的两个参数解释：

1.deliveryTag消息的编号 2.multiple是否应答之前的消息  true表示应答  false表示不应答，但是会应答当前这条消息

```java
public class Woker01 {
    private static final String QUEUE_NAME="hello";
    public static void main(String[] args) throws  Exception{
        Channel channel = RabbitMqUtils.getChannel();
        System.out.println("我是消费者a2");
        channel.basicConsume(QUEUE_NAME, false,
                //consumerTag消费的一个标记 delivery传递给消费者的传递内容
                (consumerTag, delivery) -> {
                    System.out.println("消费者a2:"+consumerTag);
                    String string = new String(delivery.getBody());
                    System.out.println("接受的消息：" + string);
                    long deliveryTag = delivery.getEnvelope().getDeliveryTag();
                    System.out.println("编号：" + deliveryTag);
                    channel.basicAck(deliveryTag,true);
                },
                consumerTag -> {
                    System.out.println(consumerTag + "消费者：取消消费消息");
                });
    }
}
```

## **消息自动重新入队** 

如果消费者由于某些原因失去连接(其通道已关闭，连接已关闭或 TCP 连接丢失)，导致消息未发送 ACK 确认，RabbitMQ 将了解到消息未完全处理，并将对其重新排队。如果此时其他消费者可以处理，它将很快将其重新分发给另一个消费者。这样，即使某个消费者偶尔死亡，也可以确保不会丢失任何消息

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220106183854.png)

**生产者:**

```java
public class Task01 {
    private static final String QUEUE_NAME="hello";
    public static void main(String[] args) throws  Exception{
        Channel channel = RabbitMqUtils.getChannel();
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        System.out.println("输入消息：");
        Scanner scanner=new Scanner(System.in);
        while (scanner.hasNext()) {
            String message = scanner.next();
            channel.basicPublish("", QUEUE_NAME, null, message.getBytes());
            System.out.println("消息发送完成");
        }
    }
}

```

**消费者1:**

```java
public class Woker02 {
    private static final String QUEUE_NAME="hello";
    public static void main(String[] args) throws  Exception{
        Channel channel = RabbitMqUtils.getChannel();
        //取消自动应答我们自己手动应答
        System.out.println("我是消费者a2处理时间长(没有处理消息)");
        channel.basicConsume(QUEUE_NAME, false,
                //consumerTag消费的一个标记 delivery传递给消费者的传递内容
                (consumerTag, delivery) -> {
                    System.out.println("消费者a1:"+consumerTag);
                    String string = new String(delivery.getBody());
                    System.out.println("接受的消息：" + string);
                    long deliveryTag = delivery.getEnvelope().getDeliveryTag();
                    System.out.println("编号：" + deliveryTag);
                    SleepUtils.sleep(30);
                },
                consumerTag -> {
                    System.out.println(consumerTag + "消费者：取消消费消息");
                });
    }
}

```



**消费者2:**

```java
public class Woker03 {
    private static final String QUEUE_NAME="hello";
    public static void main(String[] args) throws  Exception{
        Channel channel = RabbitMqUtils.getChannel();
        //取消自动应答我们自己手动应答
        System.out.println("我是消费者a3处理时间短(处理消息)");
        channel.basicConsume(QUEUE_NAME, false,
                //consumerTag消费的一个标记 delivery传递给消费者的传递内容
                (consumerTag, delivery) -> {
                    System.out.println("消费者a1:"+consumerTag);
                    String string = new String(delivery.getBody());
                    System.out.println("接受的消息：" + string);
                    long deliveryTag = delivery.getEnvelope().getDeliveryTag();
                    System.out.println("编号：" + deliveryTag);
                    SleepUtils.sleep(2);
                    channel.basicAck(deliveryTag,false);
                },
                consumerTag -> {
                    System.out.println(consumerTag + "消费者：取消消费消息");
                });
    }
}

```



## 持久化

### 	**概念**	

  刚刚我们已经看到了如何处理任务不丢失的情况，但是如何保障当 RabbitMQ 服务停掉以后消息生产者发送过来的消息不丢失。默认情况下 RabbitMQ 退出或由于某种原因崩溃时，它忽视队列和消息，除非告知它不要这样做。确保消息不会丢失需要做两件事：我们需要将队列和消息都标记为持久化。

###  **队列如何实现持久化** 

之前我们创建的队列都是非持久化的，rabbitmq 如果重启的化，该队列就会被删除掉，如果要队列实现持久化 需要在声明队列的时候把 durable 参数设置为持久化

```java
 boolean durable=true;
 channel.queueDeclare(QUEUE_NAME, false, false, false, null);
```

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220106193850.png)

但是需要注意的就是如果之前声明的队列不是持久化的，需要把原先队列先删除，或者重新创建一个持久化的队列，不然就会出现错误

以下为控制台中持久化与非持久化队列的 UI 显示区、

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220106193912.png)

这个时候即使重启 rabbitmq 队列也依然存在

### **消息实现持久化** 

要想让消息实现持久化需要在消息生产者修改代码，MessageProperties.PERSISTENT_TEXT_PLAIN 添加这个属性。

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220106193955.png)

将消息标记为持久化并不能完全保证不会丢失消息。尽管它告诉 RabbitMQ 将消息保存到磁盘，但是这里依然存在当消息刚准备存储在磁盘的时候 但是还没有存储完，消息还在缓存的一个间隔点。此时并没有真正写入磁盘。持久性保证并不强，但是对于我们的简单任务队列而言，这已经绰绰有余了.

### **不公平分发** 

在最开始的时候我们学习到 RabbitMQ 分发消息采用的轮训分发，但是在某种场景下这种策略并不是很好，比方说有两个消费者在处理任务，其中有个消费者 1 处理任务的速度非常快，而另外一个消费者 2 处理速度却很慢，这个时候我们还是采用轮训分发的化就会到这处理速度快的这个消费者很大一部分时间处于空闲状态，而处理慢的那个消费者一直在干活，这种分配方式在这种情况下其实就不太好，但是RabbitMQ 并不知道这种情况它依然很公平的进行分发。为了避免这种情况，

我们可以设置参数 channel.basicQos(1);

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220106194131.png)

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220106194218.png)

意思就是如果这个任务我还没有处理完或者我还没有应答你，你先别分配给我，我目前只能处理一个任务，然后 rabbitmq 就会把该任务分配给没有那么忙的那个空闲消费者，当然如果所有的消费者都没有完成手上任务，队列还在不停的添加新任务，队列有可能就会遇到队列被撑满的情况，这个时候就只能添加新的 worker 或者改变其他存储任务的策略。

### **预取值** 



本身消息的发送就是异步发送的，所以在任何时候，channel 上肯定不止只有一个消息另外来自消费者的手动确认本质上也是异步的。因此这里就存在一个未确认的消息缓冲区，因此希望开发人员能**限制此缓冲区的大小，以避免缓冲区里面无限制的未确认消息问题。**这个时候就可以通过使用 basic.qos 方法设置“预取计数”值来完成的。**该值定义通道上允许的未确认消息的最大数量**。一旦数量达到配置的数量，RabbitMQ 将停止在通道上传递更多消息，除非至少有一个未处理的消息被确认，例如，假设在通道上有未确认的消息 5、6、7，8，并且通道的预取计数设置为 4，此时 RabbitMQ 将不会在该通道上再传递任何消息，除非至少有一个未应答的消息被 ack。比方说 tag=6 这个消息刚刚被确认 ACK，RabbitMQ 将会感知这个情况到并再发送一条消息。消息应答和 QoS 预取值对用户吞吐量有重大影响。通常，增加预取将提高向消费者传递消息的速度。**虽然自动应答传输消息速率是最佳的，但是，在这种情况下已传递但尚未处理的消息的数量也会增加，从而增加了消费者的 RAM 消耗**(随机存取存储器)应该小心使用具有无限预处理的自动确认模式或手动确认模式，消费者消费了大量的消息如果没有确认的话，会导致消费者连接节点的内存消耗变大，所以找到合适的预取值是一个反复试验的过程，不同的负载该值取值也不同 100 到 300 范围内的值通常可提供最佳的吞吐量，并且不会给消费者带来太大的风险。预取值为 1 是最保守的。当然这将使吞吐量变得很低，特别是消费者连接延迟很严重的情况下，特别是在消费者连接等待时间较长的环境中。对于大多数应用来说，稍微高一点的值将是最佳的。

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220106200136.png)
