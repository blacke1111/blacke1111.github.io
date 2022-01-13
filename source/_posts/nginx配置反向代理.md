---
title: nginx配置反向代理
date: 2022-01-12 20:04:24
categories: nginx
---

# **反向代理实例一**

实现效果：使用 nginx 反向代理，访问 www.123.com 直接跳转到 127.0.0.1:8080

准备工作

（1）再linux系统安装tomcat，使用默认端口8080

tomcat安装文件放到linux系统中，解压

进入tomcat的bin目录中，./startup.sh启动tomcat服务器

记得**关闭防火墙**

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220112201738.png)

## 配置反向代理

### 过程:

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220112201858.png)

### **具体配置**

**第一步** 在 **windows 系统的host文件进行域名和 ip 对应关系的配置**

现在自己本机上配置host文件

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220112202522.png)

hosts文件最下面添加：

```
服务器ip www.123.com
```

**第二步 在** **nginx** **进行请求转发的配置（反向代理配置）**

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220112202626.png)

**第三步最终测试**

**重启nginx：**

```java
./nginx -s reload
```



![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220112204036.png)

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220112212255.png)
