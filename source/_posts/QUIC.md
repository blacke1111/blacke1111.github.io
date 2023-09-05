---
title: QUIC
date: 2022-07-25 22:03:49
tags: 计算机网络
---

文章推荐：https://www.yuque.com/fcant/web-notes/guo3ha#fBB3S

QUIC  基于UDP的传输层协议  

为什么不把QUIC 作为和UDP和TCP 并列的一个传输协议 而要去基于UDP 呢

因为 这样编写一个QUIC协议是十分复杂的，而且还要去修改操作系统的内核，是十分复杂的一件事，还需要把他部署到 这么世界上这么多电脑中是十分困难的。



QUIC面临的挑战



1.路由封杀UDP 443端口 这个也是QUIC部署的端口

2.UDP包过多，由于QS限定，会被服务商认为是攻击，UDP包被丢弃

3.无论是路由器还是防火墙目前对QUIC都还没有做好准备
