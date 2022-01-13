---
title: nginx负载均衡
date: 2022-01-13 19:11:32
categories: nginx
---

# 1、实现效果

（1）浏览器地址栏输入地址 http://192.168.17.129/edu/a.html，负载均衡效果，平均 8080和 8081 端口中

# 2、准备工作

（1）准备两台 tomcat 服务器，一台 8080，一台 8081
（2）在两台 tomcat 里面 webapps 目录中，创建名称是 edu 文件夹，在 edu 文件夹中创建页面 a.html，用于测试

# 3、在 nginx 的配置文件中进行负载均衡的配置

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220113192825.png)

# 测试效果：

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220113193003.png)

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220113193030.png)

# nginx 分配服务器策略

## 第一种 轮询（默认）

每个请求按时间顺序逐一分配到不同的后端服务器，如果后端服务器 down 掉，能自动剔除。

## 第二种 weight

weight 代表权重默认为 1,权重越高被分配的客户端越多

## 第三种 ip_hash

每个请求按访问 ip 的 hash 结果分配，这样每个访客固定访问一个后端服务器

## 第四种 fair（第三方）

按后端服务器的响应时间来分配请求，响应时间短的优先分配。
