---
title: nacos
date: 2022-01-24 19:53:20
categories: springcloud
---

Nacos 是一个易于使用的动态服务发现、配置和服务管理平台，用于构建云原生应用程序。

借助 Spring Cloud Alibaba Nacos Discovery，您可以快速访问基于 Spring Cloud 编程模型的 Nacos 服务注册功能。

Nacos 致力于帮助您发现、配置和管理微服务。Nacos 提供了一组简单易用的特性集，帮助您快速实现动态服务发现、服务配置、服务元数据及流量管理。

Nacos 帮助您更敏捷和容易地构建、交付和管理微服务平台。 Nacos 是构建以“服务”为中心的现代应用架构 (例如微服务范式、云原生范式) 的服务基础设施。

## 服务注册/发现：Nacos Discovery

服务发现是微服务架构中的关键组件之一。在这样的架构中，手动为每个客户端配置服务列表可能是一项艰巨的任务，并且使动态扩展变得极其困难。Nacos Discovery 帮助您将服务自动注册到 Nacos 服务器，Nacos 服务器会跟踪服务并动态刷新服务列表。此外，Nacos Discovery 将服务实例的一些元数据，如主机、端口、健康检查 URL、主页等注册到 Nacos。关于如何下载和启动 Nacos，请参考[Nacos 官网](https://nacos.io/zh-cn/docs/quick-start.html)。

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220124203001.png)

首先去官网下载最新的安装包：

windows下：

解压后在bin目录下执行：

```
startup.cmd -m standalone
```

密码账号都是nacos

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220124203218.png)

关闭：

```
shutdown.cmd
```

## 构建生产者模块：

![image-20220124203303880](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220124203303880.png)

pom：

```
 <dependencies>
        <!--SpringCloud ailibaba nacos -->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
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

yml:

```yml
server:
  port: 9001

spring:
  application:
    name: nacos-payment-provider
  cloud:
    nacos:
      discovery:
        server-addr: localhost:8848

management:
  endpoints:
    web:
      exposure:
        include: '*'
```

controller:

```java
@RestController
public class PaymentController
{
    @Value("${server.port}")
    private String serverPort;

    @GetMapping(value = "/payment/nacos/{id}")
    public String getPayment(@PathVariable("id") Integer id)
    {
        return "nacos registry, serverPort: "+ serverPort+"\t id"+id;
    }
}

```

主启动：

```java
@SpringBootApplication
@EnableDiscoveryClient
public class AlibabaProviderMain9001 {
    public static void main(String[] args) {
        SpringApplication.run(AlibabaProviderMain9001.class,args);
    }
}

```

启动：

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220124203441.png)

克隆一个模块9002。。搭建集群为负载均衡做准备

负载均衡：

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220124205324.png)

利用ribbon实现负载均衡

## 消费者module搭建：

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220124205017.png)

pom：

```
<dependencies>
        <!--SpringCloud ailibaba nacos -->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
        </dependency>
        <!-- 引入自己定义的api通用包，可以使用Payment支付Entity -->
        <dependency>
            <groupId>com.example.springcloud</groupId>
            <artifactId>cloud-api-commons</artifactId>
            <version>1.0-SNAPSHOT</version>
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

yml:

```yml
server:
  port: 83


spring:
  application:
    name: nacos-order-consumer
  cloud:
    nacos:
      discovery:
        server-addr: localhost:8848


#消费者将要去访问的微服务名称(注册成功进nacos的微服务提供者)
service-url:
  nacos-user-service: http://nacos-payment-provider

```

config:

```java
@Configuration
public class WebConfig {

    @Bean
    @LoadBalanced
    public RestTemplate getRestTemplate(){
        return  new RestTemplate();
    }
}

```

controller:

```java
@RestController
public class ConsumerController {
    @Resource
    private RestTemplate restTemplate;

    @Value("${service-url.nacos-user-service}")
    private  String serverPort;

    @GetMapping(value = "/consumer/payment/nacos/{id}")
    public String paymentInfo(@PathVariable("id")Long id){
        return  restTemplate.getForObject(serverPort+"/payment/nacos/"+id,String.class);

    }
}
```

主启动：

```java
@SpringBootApplication
@EnableDiscoveryClient
public class AlibabaOrderMain83 {
    public static void main(String[] args) {
        SpringApplication.run(AlibabaOrderMain83.class,args);
    }
}

```

启动主启动类。

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220124205209.png)

访问：http://localhost:83/consumer/payment/nacos/1

可以看到轮询访问9001和9002

## 注册中心对比：

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220124205826.png)

C是所有节点在同一时间看到的数据是一致的；而A的定义是所有的请求都会收到响应。


何时选择使用何种模式？
一般来说，
如果不需要存储服务级别的信息且服务实例是通过nacos-client注册，并能够保持心跳上报，那么就可以选择AP模式。当前主流的服务如 Spring cloud 和 Dubbo 服务，都适用于AP模式，AP模式为了服务的可能性而减弱了一致性，因此AP模式下只支持注册临时实例。

如果需要在服务级别编辑或者存储配置信息，那么 CP 是必须，K8S服务和DNS服务则适用于CP模式。
CP模式下则支持注册持久化实例，此时则是以 Raft 协议为集群运行模式，该模式下注册实例之前必须先注册服务，如果服务不存在，则会返回错误。

curl -X PUT '$NACOS_SERVER:8848/nacos/v1/ns/operator/switches?entry=serverMode&value=CP'

## 配置中心：

规则：

${prefix}-${spring.profiles.active}.${file-extension}

### 创建module：



![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220125191539.png)

### pom:

```xml
 <dependencies>
        <!--nacos-config-->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
        </dependency>
        <!--nacos-discovery-->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
        </dependency>
        <!--web + actuator-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <!--一般基础配置-->
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

### yml:

bootstrap.yml:

```yml
# nacos配置
server:
  port: 3377

spring:
  application:
    name: nacos-config-client
  cloud:
    nacos:
      discovery:
        server-addr: localhost:8848 #Nacos服务注册中心地址
      config:
        server-addr: localhost:8848 #Nacos作为配置中心地址
        file-extension: yaml #指定yaml格式的配置


# ${spring.application.name}-${spring.profile.active}.${spring.cloud.nacos.config.file-extension}
# nacos-config-client-dev.yaml
```

aplication.yml:

```yml

spring:
  profiles:
    active: dev # 表示开发环境
```

### controller：

```java
@RestController
@RefreshScope
public class ConfigClientController {

    @Value("${config.info}")
    private  String configInfo;

    @GetMapping(value = "/config/info")
    public String getConfigInfo(){
        return  configInfo;
    }
}

```

### 主启动：

```java
@EnableDiscoveryClient
@SpringBootApplication
public class NacosConfigClientMain3377
{
    public static void main(String[] args) {
        SpringApplication.run(NacosConfigClientMain3377.class, args);
    }
}
```

### nacos：

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220125191834.png)

访问http://localhost:3377/config/info

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220125191808.png)

读取顺序：

先通过bootstrap。yml配置的 config文件的地址查找 配置文件 ，如果查找得到就配置这里面的 ，如果没有自己项目中的application.yml

优先级：***bootstrap.yml > application.yml > application-dev.yml > nacos-service.yaml >nacos-service-dev.yaml***

注意：**虽然bootstrap中配置了启动环境为dev 但是在启动服务的时候依然会读取本地的application.yml和nacos上面的配置文件**

如果bootstrap.yml配置的是测试环境

```yml
 profiles:
    active: stg
```

那么服务启动后，各个配置文件的加载顺序为

**bootstrap.yml > application.yml > application-stg.yml > order-service.yaml >order-service-stg.yaml**

重点：**后面加载的配置会覆盖前面加载的配置内容**

命名空间配置要配置在bootstrap.yml中：

```properties
spring.cloud.nacos.config.namespace=a5bbe21a-6c89-47f7-9c27-ade33f803f00
```

指定配置文件bootstart：

```properties
spring.cloud.nacos.config.ext-config[0].data-id=port.properties
# 开启动态刷新配置，否则配置文件修改，工程无法感知
spring.cloud.nacos.config.ext-config[0].refresh=true
```

优先级：**bootstrap.yml > application.yml > application-stg.yml > port.properties>order-service.yaml >order-service-stg.yaml**



## 动态刷新：

nacos自带动态刷新，不需要发送post请求

## 配置分类：

是什么
   类似Java里面的package名和类名
   最外层的namespace是可以用于区分部署环境的，Group和DataID逻辑上区分两个目标对象。

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220125193431.png)

默认情况：
Namespace=public，Group=DEFAULT_GROUP, 默认Cluster是DEFAULT

Nacos默认的命名空间是public，Namespace主要用来实现隔离。
比方说我们现在有三个环境：开发、测试、生产环境，我们就可以创建三个Namespace，不同的Namespace之间是隔离的。

Group默认是DEFAULT_GROUP，Group可以把不同的微服务划分到同一个分组里面去

Service就是微服务；一个Service可以包含多个Cluster（集群），Nacos默认Cluster是DEFAULT，Cluster是对指定微服务的一个虚拟划分。
比方说为了容灾，将Service微服务分别部署在了杭州机房和广州机房，
这时就可以给杭州机房的Service微服务起一个集群名称（HZ），
给广州机房的Service微服务起一个集群名称（GZ），还可以尽量让同一个机房的微服务互相调用，以提升性能。

最后是Instance，就是微服务的实例。



默认Nacos使用嵌入式数据库实现数据的存储。所以，如果启动多个默认配置下的Nacos节点，数据存储是存在一致性问题的。
为了解决这个问题，Nacos采用了集中式存储的方式来支持集群化部署，目前只支持MySQL的存储。

## 单机模式：

在0.7版本之前，在单机模式时nacos使用嵌入式数据库实现数据的存储，不方便观察数据存储的基本情况。0.7版本增加了支持mysql数据源能力，具体的操作步骤：

- 1.安装数据库，版本要求：5.6.5+

- 2.初始化mysql数据库，数据库初始化文件：nacos-mysql.sql

- 3.修改conf/application.properties文件，增加支持mysql数据源配置（目前只支持mysql），添加mysql数据源的url、用户名和密码。

  

我使用的是mysql8.0.25

application.properties：

```properties
spring.datasource.platform=mysql

db.num=1
db.url.0=jdbc:mysql://localhost:3306/nacos_config?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&serverTimezone=UTC
db.user=root
db.password=***********

```

## 集群：

我使用的是虚拟机搭建nacos集群：



**第一步**：首先我们准备3台虚拟机：

![image-20220126234027098](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20220126234027098.png)





**第二步：**    

**三台虚拟机都要配置**

在 /usr/local/software下解压你上传的压缩包，在nacos官网下载的linux压缩包包。。

```
tar -vxf  nacos-server-2.0.4.tar.gz
```

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220126234148.png)



进入nacos目录下的conf目录：

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220126234248.png)

**第三步：**

 **只需要配一台**

我们还需要准备一台虚拟机安装mysql数据库，作为nacos的外部数据源。。对应的建表语句在nacos-mysql.sql中有对应的建表语句。。

**第四步：**

**三台虚拟机都要配置**

配置conf目录下的application.properties配置文件，添加以下内容：



```properties
spring.datasource.platform=mysql
jdbc.DriverClassName=com.mysql.cj.jdbc.Driver   //mysql8.0+ 需要添加这条配置语句  ，5.0+不需要，直接删去
# 指定数据库实例数量
db.num=1
# 第一个数据库实例地址
db.url.0=jdbc:mysql://mysql那台主机的ip:3306/nacos_config?serverTimezone=GMT%2B8&characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&serverTimezone=UTC
db.user=root
db.password=123456
```

保存退出;

**第五步：**

**三台虚拟机都要配置**

在conf目录下执行下面语句：

```
cp cluster.conf.example cluster.conf
```



编辑cluster.conf文件：

```
#2022-01-26T00:54:33.422
192.168.32.3:8848
192.168.32.4:8848
192.168.32.5:8848
```



**第六步：**

进入nacos/bin目录

执行 ./startup.sh （集群启动命令细节可看官网说明） ok...

可以看到 nacos/logs/start.out 日志文件：

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220127000424.png)

这样就启动成功了；

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220127000521.png)

关闭nacos：

nacos/bin目录下执行

```
./shutdown.sh
```





**小建议**： 虚拟机处理器还是给2把不然启动的时候太卡了 内存给个4或者3 当然这需要你的硬件要求足够     不然。。。真的真的卡的我反正启动都启动不了    
