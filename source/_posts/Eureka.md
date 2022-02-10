---
title: Eureka
date: 2022-01-15 19:11:27
tags: eureka
categories: springcloud
---



##  什么是服务治理　　

​      Spring Cloud 封装了 Netflix 公司开发的 Eureka 模块来实现服务治理

      在传统的rpc远程调用框架中，管理每个服务与服务之间依赖关系比较复杂，管理比较复杂，所以需要使用服务治理，管理服务于服务之间依赖关系，可以实现服务调用、负载均衡、容错等，实现服务发现与注册。

##   什么是服务注册与发现

Eureka采用了CS的设计架构，Eureka Server 作为服务注册功能的服务器，它是服务注册中心。而系统中的其他微服务，使用 Eureka的客户端连接到 Eureka Server并维持心跳连接。这样系统的维护人员就可以通过 Eureka Server 来监控系统中各个微服务是否正常运行。
在服务注册与发现中，有一个注册中心。当服务器启动的时候，会把当前自己服务器的信息 比如 服务地址通讯地址等以别名方式注册到注册中心上。另一方（消费者|服务提供者），以该别名的方式去注册中心上获取到实际的服务通讯地址，然后再实现本地RPC调用RPC远程调用框架核心设计思想：在于注册中心，因为使用注册中心管理每个服务与服务之间的一个依赖关系(服务治理概念)。在任何rpc远程框架中，都会有一个注册中心(存放服务地址相关信息(接口地址))

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220115191444.png)

## Eureka Server提供服务注册服务

各个微服务节点通过配置启动后，会在EurekaServer中进行注册，这样EurekaServer中的服务注册表中将会存储所有可用服务节点的信息，服务节点的信息可以在界面中直观看到。

## EurekaClient通过注册中心进行访问

是一个Java客户端，用于简化Eureka Server的交互，客户端同时也具备一个内置的、使用轮询(round-robin)负载算法的负载均衡器。在应用启动后，将会向Eureka Server发送心跳(默认周期为30秒)。如果Eureka Server在多个心跳周期内没有接收到某个节点的心跳，EurekaServer将会从服务注册表中把这个服务节点移除（默认90秒）





## 案例：idea生成eurekaServer服务注册中心

1.建module

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220115192931.png)

2.改pom

```java
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>cloud2022</artifactId>
        <groupId>com.example.springcloud</groupId>
        <version>1.0-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>cloud-eureka-server7001</artifactId>
    <dependencies>
        <!--eureka-server-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
        </dependency>
        <!-- 引入自己定义的api通用包，可以使用Payment支付Entity -->
        <dependency>
            <groupId>com.example.springcloud</groupId>
            <artifactId>cloud-api-commons</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>
        <!--boot web actuator-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <!--一般通用配置-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <scope>runtime</scope>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
        </dependency>
    </dependencies>
</project>
```



3.写yml

```yml
server:
  port: 7001

eureka:
  instance:
    hostname: localhost
  client:
    #false表示不向注册中心注册自己
    register-with-eureka: false
    #false表示自己端就是组测中心，我的职责就是维护服务实例，并不需要检索服务
    fetch-registry: false
    service-url:
      defaultZone:  http://${eureka.instance.hostname}:${server.port}/eureka/

```

4.主启动

```java
@SpringBootApplication
//表示自己就是服务注册中心
@EnableEurekaServer
public class EurekaMain7001 {
    public static void main(String[] args) {
        SpringApplication.run(EurekaMain7001.class,args);
    }
}

```



5.测试

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220115194846.png)



## 注册服务：

​	![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220115195222.png)

在这个模块下加入新的pom:

```pom
<dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```



改yal

```yaml
eureka:
  client:
    #将自己注册进EurekaServer默认ture
    register-with-eureka: true
    #是否从EurekaServer抓取已有的注册信息，默认为true。单节点无所谓，集群必须设置为true才能配合ribbon使用负载均衡
    fetch-registry: true
    service-url:
      defaultZone: http://localhost:7001/eureka
```



改主启动

```java
@SpringBootApplication
@EnableEurekaClient
public class PaymentMain8001 {
    public static void main(String[] args) {
        SpringApplication.run(PaymentMain8001.class,args);
    }
}

```

测试启动主启动类:

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220115200037.png)



## 集群配置：

**相互守望。互相监测**

Eureka7001配置yml：

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220115203441.png)

```yaml
server:
  port: 7001

eureka:
  instance:
    hostname: eureka7001.com
  client:
    #false表示不向注册中心注册自己
    register-with-eureka: false
    #false表示自己端就是组测中心，我的职责就是维护服务实例，并不需要检索服务
    fetch-registry: false
    service-url:
      defaultZone:  http://eureka7002.com:7002/eureka
```

Eureka7002配置yml：

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220115203518.png)

```yaml
server:
  port: 7002

eureka:
  instance:
    hostname: eureka7002.com
  client:
    register-with-eureka: false
    fetch-registry: false
    service-url:
      defaultZone: http://eureka7001.com:7001/eureka/
```

配置8001端口服务，和80端口服务的yml文件

修改defaultZone为下：

```yml
defaultZone: http://eureka7001:7001/eureka,http://eureka7002:7002/eureka
```

测试：

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220115203627.png)

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220115203640.png)

## 配置负载均衡：

### 首先构建一个订单消费集群：

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220116193237.png)

### 再配置我们的消费者服务：

controller:

```java
@RestController
@Slf4j
public class OrderController {
    public static final  String PAYMENT_URL="http://CLOUD-PAYMENT-SERVICE";

    @Resource
    private RestTemplate restTemplate;


    @GetMapping("/consumer/payment/create")
    public CommonResult<Payment> create(Payment payment)
    {
        return restTemplate.postForObject(PAYMENT_URL +"/payment/create",payment,CommonResult.class);
    }

    @GetMapping("/consumer/payment/get/{id}")
    public CommonResult<Payment> getPayment(@PathVariable("id") Long id)
    {
        return restTemplate.getForObject(PAYMENT_URL+"/payment/get/"+id,CommonResult.class);
    }
}


```

config:

```java
@Configuration
public class ApplicationContextConfig {
    @Bean
    @LoadBalanced
    public RestTemplate getRestTemplate(){
        return  new RestTemplate();
    }
}

```



修改服务再Eurka中的服务名称

```yml
instance-id: payment8001  # 配置服务名称
prefer-ip-address: true   #访问路径可以显示ip地址
```

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220116194939.png)

## 服务发现：



### controller中添加：

```java
 @GetMapping(value = "/payment/discovery")
    public Object discovery(){
        List<String> services = discoveryClient.getServices();
        for (String element : services) {
            log.info("---element---:{}",element);
        }
        List<ServiceInstance> instances = discoveryClient.getInstances("CLOUD-PAYMENT-SERVICE");
        for (ServiceInstance instance : instances) {
            log.info("{},{},{},{}",instance.getServiceId(),instance.getHost(),instance.getPort(),instance.getUri());
        }
        return  discoveryClient;
    }
```

### **主启动类添加注解**:

```
@EnableDiscoveryClient
```

### 测试访问

http://localhost:8001/payment/discovery

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220116200614.png)

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220116200603.png)



## Eureka的自我保护：

概述
保护模式主要用于一组客户端和Eureka Server之间存在网络分区场景下的保护。一旦进入保护模式，Eureka Server将会尝试保护其服务注册表中的信息，不再删除服务注册表中的数据，也就是不会注销任何微服务。

如果在Eureka Server的首页看到以下这段提示，则说明Eureka进入了保护模式：
EMERGENCY! EUREKA MAY BE INCORRECTLY CLAIMING INSTANCES ARE UP WHEN THEY'RE NOT. 
RENEWALS ARE LESSER THAN THRESHOLD AND HENCE THE INSTANCES ARE NOT BEING EXPIRED JUST TO BE SAFE 

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220116200726.png)



一句话：某时刻某个微服务不可用了，Eureka不会立刻清理，依旧会对该微服务的信息进行保存，属于CAP里面的Ap分支

### 为什么会产生Eureka自我保护机制？

为了防止EurekaClient可以正常运行，但是 与 EurekaServer网络不通情况下，EurekaServer不会立刻将EurekaClient服务剔除

### 什么是自我保护模式？

默认情况下，如果EurekaServer在一定时间内没有接收到某个微服务实例的心跳，EurekaServer将会注销该实例（默认90秒）。但是网络分区故障发生(延时、卡顿、拥挤)时，微服务与EurekaServer之间无法正常通信，以上行为可能变得非常危险了——因为微服务本身其实是健康的，此时本不应该注销这个微服务。Eureka通过“自我保护模式”来解决这个问题——当EurekaServer节点在短时间内丢失过多客户端时（可能发生了网络分区故障），那么这个节点就会进入自我保护模式。

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220116201821.png)

在自我保护模式中，Eureka Server会保护服务注册表中的信息，不再注销任何服务实例。它的设计哲学就是宁可保留错误的服务注册信息，也不盲目注销任何可能健康的服务实例。一句话讲解：好死不如赖活着

综上，自我保护模式是一种应对网络异常的安全保护措施。它的架构哲学是宁可同时保留所有微服务（健康的微服务和不健康的微服务都会保留）也不盲目注销任何健康的微服务。使用自我保护模式，可以让Eureka集群更加的健壮、稳定。

### 怎么禁止自我保护



在Eureka服务端：

配置yml:

```yml
eureka:
	server:
		#关闭自我保护机制，保证不可用服务被及时踢除
    	enable-self-preservation: false  
    	eviction-interval-timer-in-ms: 2000
```



客户端配置(不是完全必要)：

```yml
eureka:
	instance:
		#Eureka客户端向服务端发送心跳的时间间隔，单位为秒(默认是30秒)
    	lease-renewal-interval-in-seconds: 1
  		#Eureka服务端在收到最后一次心跳后等待时间上限，单位为秒(默认是90秒)，超时将剔除服务
    	lease-expiration-duration-in-seconds: 2
```

