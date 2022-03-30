---
title: 发布确认
date: 2022-01-06 20:04:04
categories: RabbitMQ
---

# 发布确认

## **发布确认原理**

  生产者将信道设置成 confirm 模式，一旦信道进入 confirm 模式，**所有在该信道上面发布的****消息都将会被指派一个唯一的 ID(从 1 开始)**，一旦消息被投递到所有匹配的队列之后，broker就会发送一个确认给生产者(包含消息的唯一 ID)，这就使得生产者知道消息已经正确到达目的队列了，如果消息和队列是可持久化的，那么确认消息会在将消息写入磁盘之后发出，broker 回传给生产者的确认消息中 delivery-tag 域包含了确认消息的序列号，此外 broker 也可以设置basic.ack 的 multiple 域，表示到这个序列号之前的所有消息都已经得到了处理。
  confirm 模式最大的好处在于他是异步的，一旦发布一条消息，生产者应用程序就可以在等信道返回确认的同时继续发送下一条消息，当消息最终得到确认之后，生产者应用便可以通过回调方法来处理该确认消息，如果 RabbitMQ 因为自身内部错误导致消息丢失，就会发送一条 nack 消息，生产者应用程序同样可以在回调方法中处理该 nack 消息。

## **发布确认的策略**

### **开启发布确认的方法** 

发布确认默认是没有开启的，如果要开启需要调用方法 confirmSelect，每当你要想使用发布确认，都需要在 channel 上调用该方法

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220106202806.png)

### **单个确认发布** 

  这是一种简单的确认方式，它是一种**同步确认发布**的方式，也就是发布一个消息之后只有它被确认发布，后续的消息才能继续发布,waitForConfirmsOrDie(long)这个方法只有在消息被确认的时候才返回，如果在指定时间范围内这个消息没有被确认那么它将抛出异常。
  这种确认方式有一个最大的缺点就是:**发布速度特别的慢**，因为如果没有确认发布的消息就会阻塞所有后续消息的发布，这种方式最多提供每秒不超过数百条发布消息的吞吐量。当然对于某些应用程序来说这可能已经足够了。

```java
public class PublishiConfirmsTest {
    public static  int MESSAGE_COUNT=100;
    public static void main(String[] args) throws Exception {
        publishMessageIndividual();
    }
    //单个消息发布确认
    public static void publishMessageIndividual() throws Exception {
        Channel channel = RabbitMqUtils.getChannel();
        String queueName = UUID.randomUUID().toString();
        channel.queueDeclare(queueName,false,false,false,null);
        //开启发布确认 目的是为了发送消息更加安全
        channel.confirmSelect();
        long begin = System.currentTimeMillis();
        for (int i = 0; i < MESSAGE_COUNT; i++) {
            String message="消息"+i;
            channel.basicPublish("",queueName,null,message.getBytes());
            //等待broker（代理者：交换机）确认收到消息的一个方法
            boolean b = channel.waitForConfirms();
            if (b){
                System.out.println("broker已经收到消息");
            }
        }
        long end = System.currentTimeMillis();
        System.out.println("发布"+MESSAGE_COUNT+"条单个发布确认消息,耗时"+(end-begin)+"ms");
    }
}
```

### 批量发布确认：

```java
 //批量个消息发布确认
    public static void publishMessageBatch() throws Exception {
        Channel channel = RabbitMqUtils.getChannel();
        String queueName = UUID.randomUUID().toString();
        channel.queueDeclare(queueName,false,false,false,null);
        //开启发布确认 目的是为了发送消息更加安全
        channel.confirmSelect();
        //定义一个批量的值
        int batchSize=50;
        //定义一个当前已经发送多少个未确认的消息
        int outstandingMessageCount=0;
        long begin = System.currentTimeMillis();
        for (int i = 0; i < MESSAGE_COUNT; i++) {
            String message="消息"+i;
            channel.basicPublish("",queueName,null,message.getBytes());
            outstandingMessageCount++;
            if (outstandingMessageCount==batchSize){
                boolean b = channel.waitForConfirms();
                if (b){
                    System.out.println("broker已经收到消息");
                    outstandingMessageCount=0;
                }
            }
        }
        while (outstandingMessageCount>0){
            boolean b = channel.waitForConfirms();
            if (b){
                System.out.println("broker已经收到消息");
                outstandingMessageCount=0;
            }
        }
        long end = System.currentTimeMillis();
        System.out.println("发布"+MESSAGE_COUNT+"条批量发布确认消息,耗时"+(end-begin)+"ms");
    }
```



输出结果比较：

``````````````````java
`````````````````
broker已经收到消息
broker已经收到消息
broker已经收到消息
broker已经收到消息
发布100条单个发布确认消息,耗时37ms
broker已经收到消息
broker已经收到消息
发布100条批量发布确认消息,耗时7ms

``````````````````

### **异步确认发布**:

异步确认虽然编程逻辑比上两个要复杂，但是性价比最高，无论是可靠性还是效率都没得说，他是利用回调函数来达到消息可靠性传递的，这个中间件也是通过函数回调来保证是否投递成功，下面就让我们来详细讲解异步确认是怎么实现的。

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220106205014.png)

代码演示：

```java
//异步消息发布确认
    public static void publishMessageAsync() throws Exception {
        Channel channel = RabbitMqUtils.getChannel();
        String queueName = UUID.randomUUID().toString();
        channel.queueDeclare(queueName,false,false,false,null);
        //开启发布确认 目的是为了发送消息更加安全
        channel.confirmSelect();
        /**
         * 线程安全的一个map
         * 1.将序号与消息进行关联
         * 2.只要给定序号 就会把《=当前序号的值作为一个map提取出来
         */
        ConcurrentSkipListMap<Long, String> outstandingConfirms = new ConcurrentSkipListMap<>();

        /**
         * 确认收到消息回调
         * 1 deliveryTag当前消息序号
         * 2 multiple 处理一个还是多个
         */
        ConfirmCallback ackCallback=(deliveryTag,multiple)->{
            if (multiple){
                ConcurrentNavigableMap<Long, String> confirmsPartMap = outstandingConfirms.headMap(deliveryTag);
                confirmsPartMap.clear();
            }else {
                outstandingConfirms.remove(deliveryTag);
            }
        };
        ConfirmCallback nackCallback=(deliveryTag,multiple)->{
            String s = outstandingConfirms.get(deliveryTag);
            System.out.println("序号为："+deliveryTag+"的消息"+s+"需要重新发送");
        };
        channel.addConfirmListener(ackCallback,nackCallback);

        long begin = System.currentTimeMillis();
        for (int i = 0; i < MESSAGE_COUNT; i++) {
            String message="消息"+i;
            outstandingConfirms.put(channel.getNextPublishSeqNo(),message);
            channel.basicPublish("",queueName,null,message.getBytes());
        }
        long end = System.currentTimeMillis();
        System.out.println("发布"+MESSAGE_COUNT+"条异步发布确认消息,耗时"+(end-begin)+"ms");
    }
```

