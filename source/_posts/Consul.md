---
title: Consul
date: 2022-01-17 18:03:12
tags: consul
categories: springcloud
---





## 下载：

下载consul这里给一个连接，也可以自己去官网下载

链接：https://pan.baidu.com/s/1PeWgUb19ND7zO1THkNw5KQ 
提取码：yyds

## 安装：

直接解压就可以了。

## 运行：

执行：

在对应安装包压缩目录下执行：

```
consul agent -dev
```

![image-20220117191712974](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20220117191712974.png)

这里我已经注册了一个服务，consul这是一开始启动就有的

## idea创建一个服务模块

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220117191802.png)

## 配置pom：



```pom
 <dependencies>
        <!--SpringCloud consul-server -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-consul-discovery</artifactId>
        </dependency>
        <!-- SpringBoot整合Web组件 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <!--日常通用jar包配置-->
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

## 配置yml

```yml
server:
  port: 8006

spring:
  application:
    name: consul-provider-payment
  cloud:
    consul:
      host: localhost
      port: 8500
      discovery:
        service-name: consul-provider-payment

```

## 创建主启动类：

```java
@SpringBootApplication
@EnableDiscoveryClient
public class ConsulMain8006 {
    public static void main(String[] args) {
        SpringApplication.run(ConsulMain8006.class,args);
    }
}

```

## 效果图：

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220117192022.png)



## CAP:

**Consistency(强一致性),Availability(可用),Partition tolerance(分区容错性)**

**CAP理论关注的是数据，而不是整体系统设计策略**



最多只能同时较好的满足两个。
 CAP理论的核心是：一个分布式系统不可能同时很好的满足一致性，可用性和分区容错性这三个需求，
因此，根据 CAP 原理将 NoSQL 数据库分成了满足 CA 原则、满足 CP 原则和满足 AP 原则三 大类：
CA - 单点集群，满足一致性，可用性的系统，通常在可扩展性上不太强大。
CP - 满足一致性，分区容忍必的系统，通常性能不是特别高。
AP - 满足可用性，分区容忍性的系统，通常可能对一致性要求低一些。

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220117194605.png)

**AP架构**
当网络分区出现后，为了保证可用性，系统B可以返回旧值，保证系统的可用性。
结论：违背了一致性C的要求，只满足可用性和分区容错，即AP

**CP架构**
当网络分区出现后，为了保证一致性，就必须拒接请求，否则无法保证一致性
结论：违背了可用性A的要求，只满足一致性和分区容错，即CP
