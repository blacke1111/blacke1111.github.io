---
title: Ribbon
date: 2022-01-17 19:54:49
tags: ribbon
categories: springcloud
---





## 什么是Ribbon：

Spring Cloud Ribbon是基于Netflix Ribbon实现的一套客户端       负载均衡的工具。

简单的说，Ribbon是Netflix发布的开源项目，主要功能是提供客户端的软件负载均衡算法和服务调用。Ribbon客户端组件提供一系列完善的配置项如连接超时，重试等。简单的说，就是在配置文件中列出Load Balancer（简称LB）后面所有的机器，Ribbon会自动的帮助你基于某种规则（如简单轮询，随机连接等）去连接这些机器。我们很容易使用Ribbon实现自定义的负载均衡算法。

## LB负载均衡(Load Balance)是什么

简单的说就是将用户的请求平摊的分配到多个服务上，从而达到系统的HA（高可用）。
常见的负载均衡有软件Nginx，LVS，硬件 F5等。

### Ribbon本地负载均衡客户端 VS Nginx服务端负载均衡区别

 Nginx是服务器负载均衡，客户端所有请求都会交给nginx，然后由nginx实现转发请求。即负载均衡是由服务端实现的。

 Ribbon本地负载均衡，在调用微服务接口时候，会在注册中心上获取注册信息服务列表之后缓存到JVM本地，从而在本地实现RPC远程服务调用技术。

### 集中式LB

即在服务的消费方和提供方之间使用独立的LB设施(可以是硬件，如F5, 也可以是软件，如nginx), 由该设施负责把访问请求通过某种策略转发至服务的提供方；

### 进程内LB

将LB逻辑集成到消费方，消费方从服务注册中心获知有哪些地址可用，然后自己再从这些地址中选择出一个合适的服务器。

Ribbon就属于进程内LB，它只是一个类库，集成于消费方进程，消费方通过它来获取到服务提供方的地址。

总结：Ribbon其实就是一个软负载均衡的客户端组件，它可以和其他所需请求的客户端结合使用，和Eureka的结合只是其中的一个实例



![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220117200616.png)

Ribbon在工作时分成两步
第一步先选择 EurekaServer ,它优先选择在同一个区域内负载较少的server.
第二步再根据用户指定的策略，在从server取到的服务注册列表中选择一个地址。
其中Ribbon提供了多种策略：比如轮询、随机和根据响应时间加权。



## getForEntity和getForObject区别



getForObject：返回对象为响应题的对象转化城的一个对象，就是一个json串

```java
 @GetMapping(value = "/consumer/payment/get/{id}")
    public CommonResult getPaymentById(@PathVariable("id") Long id){
        return  restTemplate.getForObject(PAYMENT_URL+"/payment/get/"+id,CommonResult.class);
    }
```



getForEntity：返回对象为responseEntity对象，包含了响应中的一些重要信息，比如响应头，响应状态码，响应体等

```java
  @GetMapping(value = "/consumer/payment/getEntity/{id}")
    public CommonResult getPaymentById2(@PathVariable("id") Long id){
        ResponseEntity<CommonResult> entity = restTemplate.getForEntity(PAYMENT_URL + "/payment/get/" + id, CommonResult.class);
        if (entity.getStatusCode().is2xxSuccessful()){
            return  entity.getBody();
        }else {
            return  new CommonResult(444,"操作失败");
        }
    }
```



## 替换掉原来的轮询算法：

### **旧版**

**spring boot:2.2.2.RELEASE**

**spring cloud:Hoxton.SR1**

**spring-cloud-starter-netflix-eureka-client：2.2.1.RELEASE**

官方文档明确给出了警告：
这个自定义配置类不能放在@ComponentScan所扫描的当前包下以及子包下，
否则我们自定义的这个配置类就会被所有的Ribbon客户端所共享，达不到特殊化定制的目的了。

https://docs.spring.io/spring-cloud-commons/docs/current/reference/html/#switching-between-the-load-balancing-algorithms

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220117220909.png)

![image-20220117213341739](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20220117213341739.png)

```java
package com.example.myrule;

import com.netflix.loadbalancer.IRule;
import com.netflix.loadbalancer.RandomRule;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MyselfRule {

    @Bean
    public IRule newRule(){
        return  new RandomRule();
    }

}

```



修改主启动类加上注解：

```java
@SpringBootApplication
@EnableEurekaClient
@RibbonClient(name = "CLOUD-PAYMENT-SERVICE",configuration= MyselfRule.class)
public class OrderMain80 {
    public static void main(String[] args) {
        SpringApplication.run(OrderMain80.class,args);
    }
}

```

**这样就改成随机轮询**

### 新版：

**springboot：2.6.2**

**springcloud: 2021.0.0**

**spring-cloud-starter-netflix-eureka-client: 3.1.0**

不再使用ribbon作为客户端负载均衡器：

使用：

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220117222317.png)

同样遵守：

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220117220909.png)

步骤一：

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220117213338.png)

```java
public class MyselfRule {

    @Bean
    ReactorLoadBalancer<ServiceInstance> randomLoadBalancer(Environment environment,
                                                            LoadBalancerClientFactory loadBalancerClientFactory) {
        String name = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new RandomLoadBalancer(loadBalancerClientFactory
                .getLazyProvider(name, ServiceInstanceListSupplier.class),
                name);
    }

}

```

步骤一：

主启动类：

```java
@SpringBootApplication
@EnableEurekaClient
@LoadBalancerClient(name = "CLOUD-PAYMENT-SERVICE",configuration= MyselfRule.class)
public class OrderMain80 {
    public static void main(String[] args) {
        SpringApplication.run(OrderMain80.class,args);
    }
}

```

效果和旧版一样。
