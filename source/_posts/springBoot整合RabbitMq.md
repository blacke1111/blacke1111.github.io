---
title: springBoot整合RabbitMq
date: 2022-01-08 21:27:17
categories: RabbitMQ
---



# <center>springBoot整合RabbitMq</center>

<!--more-->

## 创建一个springboot项目

## **添加依赖** 

```java
 <dependencies>
        <!--RabbitMQ 依赖-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-amqp</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>1.2.47</version>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </dependency>
        <!--swagger-->
        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger2</artifactId>
            <version>2.9.2</version>
        </dependency>
        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger-ui</artifactId>
            <version>2.9.2</version>
        </dependency>
        <!--RabbitMQ 测试依赖-->
        <dependency>
            <groupId>org.springframework.amqp</groupId>
            <artifactId>spring-rabbit-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
```

## **修改配置文件** 

```java
spring:
  rabbitmq:
    host: 192.168.32.3
    port: 5672
    username: admin
    password: 123456
  
```

## **队列** **TTL**

### **代码架构图** 

创建两个队列 QA 和 QB，两者队列 TTL 分别设置为 10S 和 40S，然后在创建一个交换机 X 和死信交换机 Y，它们的类型都是 direct，创建一个死信队列 QD，它们的绑定关系如下：

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220108213107.png)

### **配置交换机和队列以及它们之间的关系代码** 

```java
import org.springframework.amqp.core.*;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.HashMap;
import java.util.Map;

@Configuration
public class TtlQueueConfig {
    public  static final  String X_CHANGE_NAME="X";
    public  static final  String Y_DEAD_CHANGE_NAME="Y";
    public  static final  String QA_QUEUE_NAME="QA";
    public static final String QB_QUEUE_NAME = "QB";
    public static final String QD_QUEUE_NAME = "QD";

    //交换机X
    @Bean("xExchange")
    public DirectExchange xExchange(){
        return new  DirectExchange(X_CHANGE_NAME);
    }
    //声命交换机Y
    @Bean("yExchange")
    public DirectExchange yExchange(){
        return  new DirectExchange(Y_DEAD_CHANGE_NAME);
    }

    //声名队列QA并设置过期时间5s 和 绑定死信交换机
    @Bean("queueA")
    public Queue queueA(){
        Map<String, Object> args = new HashMap<>(3);
        //声明当前队列绑定的死信交换机
        args.put("x-dead-letter-exchange", Y_DEAD_CHANGE_NAME);
        //声明当前队列的死信路由 key
        args.put("x-dead-letter-routing-key", "YD");
        //声明队列的 TTL
        args.put("x-message-ttl", 10000);
        return QueueBuilder.durable(QA_QUEUE_NAME).withArguments(args).build();
    }

    //队列QA绑定普通交换机X
    @Bean
    public Binding queueABinding(@Qualifier("queueA")Queue queueA,@Qualifier("xExchange")DirectExchange xExchange){
        return BindingBuilder.bind(queueA).to(xExchange).with("XA");
    }
    //声名队列QB
    @Bean("queueB")
    public Queue queueB(){
        Map<String, Object> args = new HashMap<>(3);
        //声明当前队列绑定的死信交换机
        args.put("x-dead-letter-exchange", Y_DEAD_CHANGE_NAME);
        //声明当前队列的死信路由 key
        args.put("x-dead-letter-routing-key", "YD");
        //声明队列的 TTL
        args.put("x-message-ttl", 40000);
        return QueueBuilder.durable(QB_QUEUE_NAME).withArguments(args).build();
    }
    //队列QB绑定普通交换机X
    @Bean
    public Binding queueBinding(@Qualifier("queueB")Queue queueB,@Qualifier("xExchange")DirectExchange xExchange){
        return BindingBuilder.bind(queueB).to(xExchange).with("XB");
    }
    //声名队列QD
    @Bean("queueD")
    public Queue queueD(){
        return QueueBuilder.durable(QD_QUEUE_NAME).build();
    }

    //绑定QD和死信交换机Y
    @Bean
    public Binding queueDBinding(@Qualifier("queueD")Queue queueD,@Qualifier("yExchange")DirectExchange yExchange){
        return  BindingBuilder.bind(queueD).to(yExchange).with("YD");
    }

}

```

### **消息生产者代码**

```java
@Slf4j
@RestController
@RequestMapping("/ttl")
public class sendMsgController {
    @Autowired
    private RabbitTemplate rabbitTemplate;
    @GetMapping("sendMsg/{message}")
    public void sendMsg(@PathVariable String message){
        log.info("当前时间{}发送一条消息{}给两个队列", new Date(),message);
        rabbitTemplate.convertAndSend("X","XA","消息来自TTL为10s队列QA"+message);
        rabbitTemplate.convertAndSend("X","XB","消息来自TTL为40s队列QB"+message);
    }
   
}
```

### **消息消费者代码** 

```java
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.core.Message;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

@Component
@Slf4j
public class DeadLetterConsumer {
    @RabbitListener(queues = "QD")
    public void receiveD(Message message){
        String s = new String(message.getBody());
        log.info("当前时间{}，收到死信队列的消息："+s);
    }
}

```

发起一个请求 http://localhost:8080/ttl/sendMsg/xxx

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220108213449.png)

第一条消息在 10S 后变成了死信消息，然后被消费者消费掉，第二条消息在 40S 之后变成了死信消息，然后被消费掉，这样一个延时队列就打造完成了。
不过，如果这样使用的话，岂不是每增加一个新的时间需求，就要新增一个队列，这里只有 10S 和 40S两个时间选项，如果需要一个小时后处理，那么就需要增加 TTL 为一个小时的队列，如果是预定会议室然后提前通知这样的场景，岂不是要增加无数个队列才能满足需求？

## **延时队列优化**

### **代码架构图** 

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220108213551.png)

### **配置文件类代码** 

```java
import org.springframework.amqp.core.*;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.HashMap;
import java.util.Map;

@Configuration
public class MsgTtlQueueConfig {
    public  static final  String Y_DEAD_CHANGE_NAME="Y";
    private static final String QC_QUEUE_NAME = "QC";

    @Bean("queueC")
    public Queue queueC(){
        Map<String, Object> args=new HashMap<>();
        //声明当前队列绑定的死信交换机
        args.put("x-dead-letter-exchange", Y_DEAD_CHANGE_NAME);
        //声明当前队列的死信路由 key
        args.put("x-dead-letter-routing-key", "YD");
        return QueueBuilder.durable(QC_QUEUE_NAME).withArguments(args).build();
    }
    //声名队列B绑定X交换机
    @Bean
    public Binding queueCBinding(@Qualifier("queueC")Queue queueC, @Qualifier("xExchange")DirectExchange xExchange){
        return BindingBuilder.bind(queueC).to(xExchange).with("XC");
    }


}

```

**消息生产者代码** 

```java
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.core.MessagePostProcessor;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import java.util.Date;
@Slf4j
@RestController
@RequestMapping("/ttl")
public class sendMsgController {
    @Autowired
    private RabbitTemplate rabbitTemplate;
    
    @GetMapping("sendTtlMsg/{message}/{ttlTime}")
    public void sendTtlMsg(@PathVariable String message,@PathVariable String ttlTime){
        log.info("当前时间{}发送一条时长：{}消息:{}给队列QC", new Date(),ttlTime,message);
        MessagePostProcessor messagePostProcessor=correlationData->{
            correlationData.getMessageProperties().setExpiration(ttlTime);
          return   correlationData;
        };
        rabbitTemplate.convertAndSend("X","XC",message,messagePostProcessor);
    }
}

```

### 消费者同上

```java
发起请求
http://localhost:8080/ttl/sendExpirationMsg/你好 1/20000
http://localhost:8080/ttl/sendExpirationMsg/你好 2/2000
```

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220108213813.png)

看起来似乎没什么问题，但是在最开始的时候，就介绍过如果使用在消息属性上设置 TTL 的方式，消息可能并不会按时“死亡“，因为 **RabbitMQ** **只会检查第一个消息是否过期**，如果过期则丢到死信队列，**如果第一个消息的延时时长很长，而第二个消息的延时时长很短，第二个消息并不会优先得到执行**

## **Rabbitmq** **插件实现延迟队列**

### **安装插件**

在官网上下载 https://www.rabbitmq.com/community-plugins.html，下载
rabbitmq_delayed_message_exchange 插件，然后解压放置到 RabbitMQ 的插件目录。
进入 RabbitMQ 的安装目录下的 plgins 目录，执行下面命令让该插件生效，然后重启 RabbitMQ
/usr/lib/rabbitmq/lib/rabbitmq_server-3.8.8/plugins
rabbitmq-plugins enable rabbitmq_delayed_message_exchange

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220108213853.png)

**代码架构图** 

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220108213915.png)

### **配置文件类代码** 

在我们自定义的交换机中，这是一种新的交换类型，该类型消息支持延迟投递机制 消息传递后并不会立即投递到目标队列中，而是存储在 mnesia(一个分布式数据系统)表中，当达到投递时间时，才投递到目标队列中。

```java
import org.springframework.amqp.core.*;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import java.util.HashMap;
@Configuration
public class DelayedQueueConfig {
    public static  final  String DELAYED_QUEUE_NAME="delayed.queue";
    public static  final  String DELAYED_CHANGE_NAME="delayed.exchange";
    public static  final  String DELAYED_ROUTING_KEY="delayed.routingkey";

    @Bean("delayedExchange")
    public CustomExchange delayedExchange(){
        HashMap<String, Object> args = new HashMap<>();
        args.put("x-delayed-type","direct");
        return  new CustomExchange(DELAYED_CHANGE_NAME,"x-delayed-message",true,false,args);
    }

    @Bean("delayedQueue")
    public Queue delayedQueue(){
        return QueueBuilder.durable(DELAYED_QUEUE_NAME).build();
    }

    @Bean
    public Binding BindingDelayedQueue(@Qualifier("delayedQueue")Queue delayedQueue,@Qualifier("delayedExchange")CustomExchange delayedExchange){
        return BindingBuilder.bind(delayedQueue).to(delayedExchange).with(DELAYED_ROUTING_KEY).noargs();
    }
}

```

### **消息生产者代码** 

```java
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.core.MessagePostProcessor;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import java.util.Date;
@Slf4j
@RestController
@RequestMapping("/ttl")
public class sendMsgController {
    @Autowired
    private RabbitTemplate rabbitTemplate;
   

    @GetMapping("sendDelayMsg/{message}/{delayTime}")
    public void sendDelayMsg(@PathVariable String message,@PathVariable Integer delayTime){
        log.info("当前时间{}发送一条延迟：{}消息:{}给队列delayed.queue", new Date(),delayTime,message);
        MessagePostProcessor messagePostProcessor=correlationData->{
            correlationData.getMessageProperties().setDelay(delayTime);
            return   correlationData;
        };
        rabbitTemplate.convertAndSend("delayed.exchange","delayed.routingkey",message,messagePostProcessor);
    }
}

```





### **消息消费者代码** :

```java
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.core.Message;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

import java.util.Date;

@Component
@Slf4j
public class DeadLetterConsumer {
    @RabbitListener(queues = "QD")
    public void receiveD(Message message){
        String s = new String(message.getBody());
        log.info("当前时间{}，收到死信队列的消息：",new Date(),s);
    }
}

```



发起请求：

```java
http://localhost:8080/ttl/sendDelayMsg/delayed你好!!/20000
http://localhost:8080/ttl/sendDelayMsg/delayed你好!!/2000
```

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220108221325.png)

第二个消息被先消费掉了，符合预期

## 总结：

延时队列在需要延时处理的场景下非常有用，使用 RabbitMQ 来实现延时队列可以很好的利用RabbitMQ 的特性，如：消息可靠发送、消息可靠投递、死信队列来保障消息至少被消费一次以及未被正确处理的消息不会被丢弃。另外，通过 RabbitMQ 集群的特性，可以很好的解决单点故障问题，不会因为单个节点挂掉导致延时队列不可用或者消息丢失。
当然，延时队列还有很多其它选择，比如利用 Java 的 DelayQueue，利用 Redis 的 zset，利用 Quartz或者利用 kafka 的时间轮，这些方式各有特点,看需要适用的场景
