---
title: openFeign
date: 2022-01-18 19:04:25
tags: openFeign
categories: springcloud
---



## openFeign是什么：

[Feign](https://github.com/OpenFeign/feign)是一个声明式 Web 服务客户端。它使编写 Web 服务客户端更容易。要使用 Feign，请创建一个接口并对其进行注释。它具有可插入的注释支持，包括 Feign 注释和 JAX-RS 注释。Feign 还支持可插拔的编码器和解码器。Spring Cloud 添加了对 Spring MVC 注释的支持，并支持使用`HttpMessageConverters`Spring Web 中默认使用的注释。Spring Cloud 集成了 Ribbon 和 Eureka，Spring Cloud CircuitBreaker，以及 Spring Cloud LoadBalancer，在使用 Feign 时提供负载均衡的 http 客户端。



## Feign能干什么

Feign旨在使编写Java Http客户端变得更容易。
前面在使用Ribbon+RestTemplate时，利用RestTemplate对http请求的封装处理，形成了一套模版化的调用方法。但是在实际开发中，由于对服务依赖的调用可能不止一处，往往一个接口会被多处调用，所以通常都会针对每个微服务自行封装一些客户端类来包装这些依赖服务的调用。所以，Feign在此基础上做了进一步封装，由他来帮助我们定义和实现依赖服务接口的定义。在Feign的实现下，我们只需创建一个接口并使用注解的方式来配置它(以前是Dao接口上面标注Mapper注解,现在是一个微服务接口上面标注一个Feign注解即可)，即可完成对服务提供方的接口绑定，简化了使用Spring cloud Ribbon时，自动封装服务调用客户端的开发量。

## Feign集成了Ribbon

利用Ribbon维护了Payment的服务列表信息，并且通过轮询实现了客户端的负载均衡。而与Ribbon不同的是，通过feign只需要定义服务绑定接口且以声明式的方法，优雅而简单的实现了服务调用

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220118203209.png)



## 案例：

### 创建module

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220118203303.png)

### pom编写:

```
 <dependencies>
        <!--openfeign-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
        </dependency>
        <!--eureka client-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <!-- 引入自己定义的api通用包，可以使用Payment支付Entity -->
        <dependency>
            <groupId>com.example.springcloud</groupId>
            <artifactId>cloud-api-commons</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>
<!--        spring boot-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
        </dependency>
<!--        日志lombok-->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </dependency>
    </dependencies>
```



### yml文件编写:

```yml
server:
  port: 80


eureka:
  client:
    register-with-eureka: false
    service-url:
      defaultZone: http://eureka7001.com:7001/eureka,http://eureka7002.com:7002/eureka
```



### 主启动类：

@EnableFeignClients(basePackages = "com.example.springcloud.service")这个开启@FeignClients

```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients(basePackages = "com.example.springcloud.service")
public class OrderFeignMain80 {
    public static void main(String[] args) {
        SpringApplication.run(OrderFeignMain80.class,args);
    }
}

```

### Service接口编写：

```java
@Component
@FeignClient("CLOUD-PAYMENT-SERVICE")   //注册在Eureka服务上的名称
public interface PaymentFeignService {
    @RequestMapping(method = RequestMethod.GET,value = "/payment/get/{id}")
    CommonResult getPaymentById(@PathVariable("id")Long id);
}

```

### Controller编写：

```java
@RestController
@Slf4j
public class OrderFeignController {

    @Resource
    private PaymentFeignService feignService;


    @GetMapping(value = "/consumer/payment/feign/get/{id}")
    public CommonResult getPaymentID(@PathVariable("id")Long id){
        return  feignService.getPaymentById(id);
    }
}
```

启动主启动类就可以了：

## 配置超时：



以往版本：

```yml
#设置feign客户端超时时间(OpenFeign默认支持ribbon)
ribbon:
#指的是建立连接所用的时间，适用于网络状况正常的情况下,两端连接所用的时间
  ReadTimeout: 5000
#指的是建立连接后从服务器读取到可用资源所用的时间
  ConnectTimeout: 5000
```



spring-cloud-starter-openfeign最新已经不支持ribbon了

```yml
feign:
  client:
    config:
      default:
        #指的是建立连接所用的时间，适用于网络状况正常的情况下,两端连接所用的时间
        ConnectTimeout: 1000
        #指的是建立连接后从服务器读取到可用资源所用的时间
        ReadTimeout: 1000
```



## 日志：

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220118203859.png)

```java
@Configuration
public class FeignConfig
{
    @Bean
    Logger.Level feignLoggerLevel()
    {
        return Logger.Level.FULL;
    }
}
```

还需要配置yml文件：

```yml

logging:
  level:
    # feign日志以什么级别监控哪个接口
    com.example.springcloud.service.PaymentFeignService: debug

```

