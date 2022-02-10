---
title: MQ的相关概念
date: 2022-01-05 21:25:10
categories: RabbitMQ
---



## 什么是MQ

MQ(message queue)，从字面意思上看，本质是个队列，FIFO 先入先出，只不过队列中存放的内容是message 而已，还是一种跨进程的通信机制，用于上下游传递消息。在互联网架构中，MQ 是一种非常常见的上下游“逻辑解耦+物理解耦”的消息通信服务。使用了 MQ 之后，消息发送上游只需要依赖 MQ，不用依赖其他服务

## **为什么要用 MQ** 

1 **流量高峰**

主要为了防止超多的请求访问服务器导致服务器崩溃。

2 **应用解耦**

解耦各个系统之间的调用，直接使用消息队列存储信息，系统只管取和读

3 异步处理

## **MQ 的分类** 

1.ActiveMQ优点：单机吞吐量万级，时效性 ms 级，可用性高，基于主从架构实现高可用性，消息可靠性较
低的概率丢失数据
缺点:官方社区现在对 ActiveMQ 5.x **维护越来越少，高吞吐量场景较少使用**。
2.Kafka
大数据的杀手锏，谈到大数据领域内的消息传输，则绕不开 Kafka，这款为大数据而生的消息中间件，以其百万级 TPS 的吞吐量名声大噪，迅速成为大数据领域的宠儿，在数据采集、传输、存储的过程中发挥着举足轻重的作用。目前已经被 LinkedIn，Uber, Twitter, Netflix 等大公司所采纳。
优点: 性能卓越，单机写入 TPS 约在百万条/秒，最大的优点，就是吞吐量高。时效性 ms 级可用性非常高，kafka 是分布式的，一个数据多个副本，少数机器宕机，不会丢失数据，不会导致不可用,消费者采用 Pull 方式获取消息, 消息有序, 通过控制能够保证所有消息被消费且仅被消费一次;有优秀的第三方Kafka Web 管理界面 Kafka-Manager；在日志领域比较成熟，被多家公司和多个开源项目使用；

功能支持：功能较为简单，主要支持简单的 MQ 功能，在大数据领域的实时计算以及日志采集被大规模使用
缺点：Kafka 单机超过 64 个队列/分区，Load 会发生明显的飙高现象，队列越多，load 越高，发送消
息响应时间变长，使用短轮询方式，实时性取决于轮询间隔时间，消费失败不支持重试；支持消息顺序，
但是一台代理宕机后，就会产生消息乱序，社区更新较慢； 

3.RocketMQ
RocketMQ 出自阿里巴巴的开源产品，用 Java 语言实现，在设计时参考了 Kafka，并做出了自己的一些改进。被阿里巴巴广泛应用在订单，交易，充值，流计算，消息推送，日志流式处理，binglog 分发等场景。
优点:单机吞吐量十万级,可用性非常高，分布式架构,消息可以做到 0 丢失,MQ 功能较为完善，还是分布式的，扩展性好,支持 10 亿级别的消息堆积，不会因为堆积导致性能下降,源码是 java 我们可以自己阅读源码，定制自己公司的 MQ
缺点：支持的客户端语言不多，目前是 java 及 c++，其中 c++不成熟；社区活跃度一般,没有在 MQ核心中去实现 JMS 等接口,有些系统要迁移需要修改大量代码
4.RabbitMQ

2007 年发布，是一个在 AMQP(高级消息队列协议)基础上完成的，可复用的企业消息系统，是当前最主流的消息中间件之一。
优点:由于 **erlang** 语言的高并发特性，性能较好；吞吐量到万级，MQ 功能比较完备,健壮、稳定、易用、跨平台、支持多种语言 如：Python、Ruby、.NET、Java、JMS、C、PHP、ActionScript、XMPP、STOMP等，支持 AJAX 文档齐全；开源提供的管理界面非常棒，用起来很好用,社区活跃度高；更新频率相当高

## MQ的选择

1.Kafka
Kafka 主要特点是基于 Pull 的模式来处理消息消费，追求高吞吐量，一开始的目的就是用于日收集和传输，适合产生**大量数据**的互联网服务的数据收集业务。**大型公司**建议可以选用，如果有**日志采集**功能肯定是首选 kafka 了。

2.RocketMQ
天生为**金融互联网**领域而生，对于可靠性要求很高的场景，尤其是电商里面的订单扣款，以及业务削峰，在大量交易涌入时，后端可能无法及时处理的情况。RoketMQ 在稳定性上可能更值得信赖，这些业务场景在阿里双 11 已经经历了多次考验，如果你的业务有上述并发场景，建议可以选择 RocketMQ。

3.RabbitMQ
结合 erlang 语言本身的并发优势，性能好**时效性微秒级**，**社区活跃度也比较高**，管理界面起来十分方便，如果你的**数据量没有那么大**，中小型公司优先选择功能比较完备的 RabbitMQ。



## RabbitMQ 的概念
RabbitMQ 是一个消息中间件：它接受并转发消息。你可以把它当做一个快递站点，当你要发送一个包裹时，你把你的包裹放到快递站，快递员最终会把你的快递送到收件人那里，按照这种逻辑 RabbitMQ 是一个快递站，一个快递员帮你传递快件。RabbitMQ 与快递站的主要区别在于，它不处理快件而是接收，存储和转发消息数据。

## **四大核心概念** 

**生产者**
产生数据发送消息的程序是生产者

**交换机**
交换机是 RabbitMQ 非常重要的一个部件，一方面它接收来自生产者的消息，另一方面它将消息推送到队列中。交换机必须确切知道如何处理它接收到的消息，是将这些消息推送到特定队列还是推送到多个队列，亦或者是把消息丢弃，这个得有交换机类型决定

**队列**
队列是 RabbitMQ 内部使用的一种数据结构，尽管消息流经 RabbitMQ 和应用程序，但它们只能存储在队列中。队列仅受主机的内存和磁盘限制的约束，本质上是一个大的消息缓冲区。许多生产者可以将消息发送到一个队列，许多消费者可以尝试从一个队列接收数据。这就是我们使用队列的方式

**消费者**
消费与接收具有相似的含义。消费者大多时候是一个等待接收消息的程序。请注意生产者，消费者和消息中间件很多时候并不在同一机器上。同一个应用程序既可以是生产者又是可以是消费者。



## **RabbitMQ 核心部分** 

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220105220852.png)



## **各个名词介绍** 

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220105220559.png)

**Broker**：接收和分发消息的应用，RabbitMQ Server 就是 Message Broker

**Virtual host**：出于多租户和安全因素设计的，把 AMQP 的基本组件划分到一个虚拟的分组中，类似

于网络中的 namespace 概念。当多个不同的用户使用同一个 RabbitMQ server 提供的服务时，可以划分出多个 vhost，每个用户在自己的 vhost 创建 exchange／queue 等

**Connection**：publisher／consumer 和 broker 之间的 TCP 连接

**Channel**：如果每一次访问 RabbitMQ 都建立一个 Connection，在消息量大的时候建立 TCP Connection 的开销将是巨大的，效率也较低。Channel 是在 connection 内部建立的逻辑连接，如果应用程序支持多线程，通常每个 thread 创建单独的 channel 进行通讯，AMQP method 包含了 channel id 帮助客户端和 message broker 识别 channel，所以 channel 之间是完全隔离的。******Channel 作为轻量级的Connection极大减少了操作系统建立** **TCP connection** **的开销** 

**Exchange**：message 到达 broker 的第一站，根据分发规则，匹配查询表中的 routing key，分发消息到 queue 中去。常用的类型有：direct (point-to-point), topic (publish-subscribe) and fanout (multicast)

**Queue**：消息最终被送到这里等待 consumer 取走

**Binding**：exchange 和 queue 之间的虚拟连接，binding 中可以包含 routing key，Binding 信息被保存到 exchange 中的查询表中，用于 message 的分发依据

## 安装

1.官网地址

	https://www.rabbitmq.com/download.html

2.文件上传

	上传到/usr/local/software 目录下(如果没有 software 需要自己创建)

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220105223800.png)

3.安装文件(分别按照以下顺序安装)

	rpm -ivh erlang-21.3-1.el7.x86_64.rpm
	
	yum install socat -y
	
	rpm -ivh rabbitmq-server-3.8.8-1.el7.noarch.rpm

3.常用命令(按照以下顺序执行)

添加开机启动 RabbitMQ 服务

	chkconfig rabbitmq-server on

启动服务

	/sbin/service rabbitmq-server start 

查看服务状态

	/sbin/service rabbitmq-server status

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220105223850.png)

停止服务(选择执行)

	/sbin/service rabbitmq-server stop 

开启 web 管理插件

	rabbitmq-plugins enable rabbitmq_management

这里访问不到的另一个原因可能是linux的防火墙没有关闭

用默认账号密码(guest)访问地址 http://47.115.185.244:15672/出现权限问题

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220105223912.png)

4.添加一个新的用户

创建账号

```
rabbitmqctl add_user admin 123456
```

设置用户角色

	rabbitmqctl set_user_tags admin administrator

设置用户权限

	set_permissions [-p <vhostpath>] <user> <conf> <write> <read>
	
	rabbitmqctl set_permissions -p "/" admin ".*" ".*" ".*"

用户 user_admin 具有/vhost1 这个 virtual host 中所有资源的配置、写、读权限当前用户和角色

	rabbitmqctl list_users

5.再次利用 admin 用户登录

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220105223942.png)

6. 重置命令

关闭应用的命令为

	rabbitmqctl stop_app

清除的命令为

	rabbitmqctl reset

重新启动命令为

	rabbitmqctl start_app
