---
title: Hystrix
date: 2022-01-18 20:40:11
tags: Hystrix
categories: springcloud
---

## 服务雪崩

多个微服务之间调用的时候，假设微服务A调用微服务B和微服务C，微服务B和微服务C又调用其它的微服务，这就是所谓的“扇出”。如果扇出的链路上某个微服务的调用响应时间过长或者不可用，对微服务A的调用就会占用越来越多的系统资源，进而引起系统崩溃，所谓的“雪崩效应”.

对于高流量的应用来说，单一的后端依赖可能会导致所有服务器上的所有资源都在几秒钟内饱和。比失败更糟糕的是，这些应用程序还可能导致服务之间的延迟增加，备份队列，线程和其他系统资源紧张，导致整个系统发生更多的级联故障。这些都表示需要对故障和延迟进行隔离和管理，以便单个依赖关系的失败，不能取消整个应用程序或系统。

## hystrix是什么：

Hystrix是一个用于处理分布式系统的延迟和容错的开源库，在分布式系统里，许多依赖不可避免的会调用失败，比如超时、异常等，Hystrix能够保证在一个依赖出问题的情况下，不会导致整体服务失败，避免级联故障，以提高分布式系统的弹性。

“断路器”本身是一种开关装置，当某个服务单元发生故障之后，通过断路器的故障监控（类似熔断保险丝），向调用方返回一个符合预期的、可处理的备选响应（FallBack），而不是长时间的等待或者抛出调用方无法处理的异常，这样就保证了服务调用方的线程不会被长时间、不必要地占用，从而避免了故障在分布式系统中的蔓延，乃至雪崩。



它能够实现**服务降级，服务熔断，接近实时的监控。。。。**



## Hystrix的重要概念：

### 服务降级：

服务器忙，请稍后尝试，不要让客户端等待并立刻返回一个友好的提示，fallback

### 服务熔断：

类比保险丝达到最大服务访问后，直接拒绝访问，拉闸限电，然后调用服务降级的方法并返回友好提示，服务的降级->进而熔断->恢复调用链路

### 服务限流：

秒杀高并发等操作，严禁—窝蜂的过来拥挤，大家排队，一秒钟N个，有序进行



## 案例

构建新项目：

### cloud-provider-hystrix-payment8001

### 修改pom文件：

```
  <dependencies>
        <!--hystrix-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
        </dependency>
        <!--eureka client-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <!--web-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency><!-- 引入自己定义的api通用包，可以使用Payment支付Entity -->
            <groupId>com.example.springcloud</groupId>
            <artifactId>cloud-api-commons</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <scope>runtime</scope>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
```

### 修改yml文件

```yml
server:
  port: 8001
spring:
  application:
    name: cloud-provider-hystrix-payment
eureka:
  client:
    register-with-eureka: true
    fetch-registry: true
    service-url:
      defaultZone: http://eureka7001.com:7001/eureka,http://eureka7002.com:7002/eureka
#      defaultZone: http://eureka7001.com:7001/eureka
```



### 创建著启动类：

```java
@SpringBootApplication
@EnableDiscoveryClient
public class PaymentHystrixMain8001 {
    public static void main(String[] args) {
        SpringApplication.run(PaymentHystrixMain8001.class,args);
    }
}
```

### 业务实现类(创建的接口为PaymentService实现他的方法)：

```java
@Service
public class PaymentServiceImpl  implements PaymentService {
    @Override
    public String paymentInfo(Integer id) {
        return "线程池： "+Thread.currentThread().getName()+" paymentInfo_OK.id:  "+id+"\t"+"哈哈";
    }
    @Override
    public String paymentError(Integer id) {
        int timeNumber=3;
        try {
            TimeUnit.SECONDS.sleep(timeNumber);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "线程池： "+Thread.currentThread().getName()+" paymentError.id:  "+id+"\t"+"哈哈,耗时(s):"+timeNumber;
    }
}

```



### Controller：

```java
@RestController
@Slf4j
public class PaymentController {
    @Resource
    private PaymentService paymentService;
    @Value("${server.port}")
    private  String serverPort;
    @GetMapping(value = "/payment/hystrix/ok/{id}")
    public String paymentInfo_OK(@PathVariable("id")Integer id){
        String s = paymentService.paymentInfo(id);
        log.info("***result:{}",s);
        return s;
    }
    @GetMapping(value = "/payment/hystrix/error/{id}")
    public String paymentError(@PathVariable("id")Integer id){
        String s = paymentService.paymentError(id);
        log.info("***result:{}",s);
        return s;
    }
}
```

启动主启动类：



开启Jmeter，来20000个并发压死8001,20000个请求都去访问paymentlnfo TimeOut服务

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220119193626.png)

我们本来不需要耗时的服务变得卡顿原因：

tomcat的默认的工作线程数被打满 了，没有多余的线程来分解压力和处理。

结论：上面还是服务提供者8001自己测试，假如此时夕部的消费者80也来访问，那消费者只能干等，最终导致消费端80不满意，服务端8001直接被拖死

再构建一个消费者服务**cloud-consumer-feign-hystrix-order80**使用feign实现上的的两个服务接口。

正因为有上述故障或不佳表现才有我们的降级/容错/限流等技术诞生



## 服务降级配置：

### 8001fallback：

业务类启用：

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220119194143.png)

主启动激活：

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220119194236.png)

### 80fallback：

业务类启用：

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220119194412.png)

主启动激活：

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220119194504.png)



### 目前问题：

#### 每个业务方法对应一个兜底的方法，代码膨胀

**解决办法：**增加一个默认的兜底方法

修改业务类：

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220119195054.png)

#### 统一和定义的分开解耦

解决办法：

修改yml：

```yml
feign:
  hystrix:
    enabled: true
```

根据cloud-consumer-feign-hystrix-order80已经有的PaymentHystrixService接口,重新新建一个类(PaymentFallbackService)实现该接口，统一为接口里面的方法进行异常处理

```java
@Service
public class PaymentFallbackService implements  PaymentHystrixService{

    @Override
    public String paymentInfo_OK(Integer id) {
        return "服务调用失败，提示来自：paymentInfo_OK";
    }

    @Override
    public String paymentError(Integer id) {
        return "服务调用失败，提示来自：paymentError";
    }
}

```



![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220119195343.png)



## 服务熔断：

一句话就是家里的保险丝

### 熔断机制概述

熔断机制是应对雪崩效应的一种微服务链路保护机制。当扇出链路的某个微服务出错不可用或者响应时间太长时，会进行服务的降级，进而熔断该节点微服务的调用，快速返回错误的响应信息。
当检测到该节点微服务调用响应正常后，**恢复调用链路。**

在Spring Cloud框架里，熔断机制通过Hystrix实现。Hystrix会监控微服务间调用的状况，当失败的调用到一定阈值，缺省是5秒内20次调用失败，就会启动熔断机制。熔断机制的注解是@HystrixCommand。

再原先的服务8001fallback降级上添加服务熔断：

```java
@Service
public class PaymentServiceImpl  implements PaymentService {
    @Override
    public String paymentInfo(Integer id) {
        return "线程池： "+Thread.currentThread().getName()+" paymentInfo_OK.id:  "+id+"\t"+"哈哈";
    }

    @HystrixCommand(fallbackMethod = "paymentInfo_TimeOutHandler",commandProperties = {
            @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds",value = "3000")
    })
    @Override
    public String paymentError(Integer id) {
        int timeNumber=3;
        try {
            TimeUnit.SECONDS.sleep(timeNumber);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "线程池： "+Thread.currentThread().getName()+" paymentError.id:  "+id+"\t"+"哈哈,耗时(s):"+timeNumber;
    }

    public String    paymentInfo_TimeOutHandler(Integer id){

        return "线程池： "+Thread.currentThread().getName()+" paymentInfo_TimeOutHandler.id:  "+id+"\t"+"哈哈,:o(Π_Π)0";

    }

    //==服务熔断
    //=========服务熔断
    @HystrixCommand(fallbackMethod = "paymentCircuitBreaker_fallback",commandProperties = {
            @HystrixProperty(name = "circuitBreaker.enabled",value = "true"), //是否看开启断路器
            @HystrixProperty(name = "circuitBreaker.requestVolumeThreshold",value = "10"),  //请求次数
            @HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds",value = "10000"), //时间窗口期
            @HystrixProperty(name = "circuitBreaker.errorThresholdPercentage",value = "60"), //失败率达到多少后跳闸
    })
    public String paymentCircuitBreaker(@PathVariable("id") Integer id)
    {
        if(id < 0)
        {
            throw new RuntimeException("******id 不能负数");
        }
        String serialNumber = IdUtil.simpleUUID();

        return Thread.currentThread().getName()+"\t"+"调用成功，流水号: " + serialNumber;
    }
    public String paymentCircuitBreaker_fallback(@PathVariable("id") Integer id)
    {
        return "id 不能负数，请稍后再试，/(ㄒoㄒ)/~~   id: " +id;
    }
}

```

