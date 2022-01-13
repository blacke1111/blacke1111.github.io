---
title: nginx动静分离
date: 2022-01-13 19:37:37
categories: nginx
---

## **什么是动静分离**

Nginx 动静分离简单来说就是把动态跟静态请求分开，不能理解成只是单纯的把动态页面和静态页面物理分离。严格意义上说应该是动态请求跟静态请求分开，可以理解成使用 Nginx 处理静态页面，Tomcat 处理动态页面。动静分离从目前实现角度来讲大致分为两种，一种是纯粹把静态文件独立成单独的域名，放在独立的服务器上，也是目前主流推崇的方案；另外一种方法就是动态跟静态文件混合在一起发布，通过 nginx 来分开。通过 location 指定不同的后缀名实现不同的请求转发。通过 expires 参数设置，可以使浏览器缓存过期时间，减少与服务器之前的请求和流量。具体 Expires 定义：是给一个资源设定一个过期时间，也就是说无需去服务端验证，直接通过浏览器自身确认是否过期即可，所以不会产生额外的流量。此种方法非常适合不经常变动的资源。（如果经常更新的文件，不建议使用 Expires 来缓存），我这里设置 3d，表示在这 3 天之内访问这个 URL，发送一个请求，比对服务器该文件最后更新时间没有变化，则不会从服务器抓取，返回状态码304，如果有修改，则直接从服务器重新下载，返回状态码 200。



## 1.项目资源准备

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220113195854.png)

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220113195913.png)

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220113195928.png)

## 2.进行 nginx 配置

找到 nginx 安装目录，打开/conf/nginx.conf 配置文件，

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220113195225.png)

测试结果：

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220113195719.png)

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220113195658.png)
