---
title: nginx原理
date: 2022-01-13 22:42:55
categories: nginx
---

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220113224412.png)

**2worker如何进行工作的**

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220113224439.png)

3、一个 master 和多个 woker 有好处
**（1）可以使用 nginx –s reload 热部署，利用 nginx 进行热部署操作**
**（2）每个 woker 是独立的进程，如果有其中的一个 woker 出现问题，其他 woker 独立的，**
继续进行争抢，实现请求过程，不会造成服务中断
4、设置多少个 woker 合适

​	**worker** **数和服务器的** **cpu** **数相等是最为适宜的**

5、连接数 worker_connection
	第一个：发送请求，占用了 woker 的几个连接数？
		**2 或者 4 个**
    第二个：nginx 有一个 master，有四个 woker，每个 woker 支持最大的连接数 1024，支持的最大并发数是多少？
		**普通的静态访问最大并发数是： worker_connections * worker_processes /2，  而如果是 HTTP 作 为反向代理来说，最大并发数量应该是 worker_connections * worker_processes/4。**
