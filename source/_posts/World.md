---

title: Hello World
date: 2022-01-05 22:44:51
categories: RabbitMQ
---

在本教程的这一部分中，我们将用 Java 编写两个程序。发送单个消息的生产者和接收消息并打印
出来的消费者。我们将介绍 Java API 中的一些细节。

在下图中，“ P”是我们的生产者，“ C”是我们的消费者。中间的框是一个队列-RabbitMQ 代表使用者保留的消息缓冲区

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220105224634.png)



使用IDEA创建一个Maven项目

## 1.导入依赖

```java
 <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <source>8</source>
                    <target>8</target>
                </configuration>
            </plugin>
        </plugins>
    </build>
    <dependencies>
        <!--rabbitmq 依赖客户端-->
        <dependency>
            <groupId>com.rabbitmq</groupId>
            <artifactId>amqp-client</artifactId>
            <version>5.8.0</version>
        </dependency>
        <!--操作文件流的一个依赖-->
        <dependency>
            <groupId>commons-io</groupId>
            <artifactId>commons-io</artifactId>
            <version>2.6</version>
        </dependency>
    </dependencies>
```

## 2.**消息生产者**

```java
public class Producer {
    private static final String QUEUE_NAME="hello";
    public static void main(String[] args) {
        //1创建一个连接工厂
        ConnectionFactory factory=new ConnectionFactory();
        factory.setHost("192.168.32.3");
        factory.setUsername("admin");
        factory.setPassword("123456");
        try {
//            创建一个来连接
            Connection connection = factory.newConnection();
            //创建一个Channel
            Channel channel = connection.createChannel();
            //创建一个队列
            /**
             * 1.队列名称
             * 2.durable队列是否持久化  true 代表持久化
             * 3.exclusive  是否只让一个消费者独自消费该队列消息
             * 4.autoDelete消费者断开连接是否自动删除该队列  true代表自动删除
             * 5.其他参数
             */
            channel.queueDeclare(QUEUE_NAME,false,false,false,null);
            /*
                1.交换机的名称
                2.路由的key
                    默认采用队列名称作为路由key，该消息就会路由到队列中
                3.其他参数配置
                4.发送消息内容
             */
            String message="hello world";
            channel.basicPublish("",QUEUE_NAME,null,message.getBytes());
            System.out.println("消息发送完成");
            channel.close();
            connection.close();
        } catch (IOException e) {
            e.printStackTrace();
        } catch (TimeoutException e) {
            e.printStackTrace();
        }

    }
}
```



## 3.**消息消费者**

```java
public class Consumer {
    private static final String QUEUE_NAME="hello";
    public static void main(String[] args) {
        ConnectionFactory factory=new ConnectionFactory();
        factory.setHost("192.168.32.3");
        factory.setUsername("admin");
        factory.setPassword("123456");
        System.out.println("消费者启动等待消息.............");
        try {
            Connection connection = factory.newConnection();
            Channel channel = connection.createChannel();
            /**
             * 消费者消费队列里面的消息
             * 1.消费那个队列
             * 2.是否采用自动应答
             * 3.队列推送消息给消费者的时候，如何消费消息
             * 4.消费者取消消费的回调接口
             */
            channel.basicConsume(QUEUE_NAME,true,
                    //consumerTag消费的一个标记 delivery传递给消费者的传递内容
                    (consumerTag, delivery) -> {
                        System.out.println(consumerTag);
                        String string=new String(delivery.getBody());
                        System.out.println("接受的消息："+string);
                        long deliveryTag = delivery.getEnvelope().getDeliveryTag();
                        System.out.println("编号："+deliveryTag);

                    },
                    consumerTag -> {
                        System.out.println(consumerTag+"消费者：取消消费消息");
                    });
        } catch (IOException e) {
            e.printStackTrace();
        } catch (TimeoutException e) {
            e.printStackTrace();
        }
    }
}
```

