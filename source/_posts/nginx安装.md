---
title: nginx安装
date: 2022-01-12 19:03:52
categories: nginx
---



# 1.进入nginx官网下载

nginx压缩包

我这里提供了下载链接：

链接：https://pan.baidu.com/s/1afOXqmdeerpA2Y-hPvGJWA 
提取码：lxrs

# 2.安装依赖包pcre

（1）首先把要压缩包放入linux的 /usr/prce下

（2）使用如下命令解压压缩包

```java
tar -xvf pcre-8.37.tar.gz 
```

（3）进入解压目录 ，执行

```java
./configure
```

(4)执行

```java
make&&make install
```

(5)查看是否安装成功

```java
pcre-config --version
```

# 3.安装其他的依赖

```java
yum -y install make zlib zlib-devel gcc-c++ libtool openssl openssl-deve
```

# 4.安装nginx

这个和第二基本类似

1.解压nginx压缩包

```java
tar -xvf nginx-1.12.2.tar.gz
```

2。进入解压压缩目录，执行

```java
./configure
```

3执行命令

```java
make && make installl
```

* **安装成功之后 ，会在/usr/local文件夹下多出一个nginx文件夹 在nginx文件夹里面有sbin启动脚本**

**查看开放的端口号**

firewall-cmd --list-all

**设置开放的端口号**

firewall-cmd --add-service=http –permanent

sudo firewall-cmd --add-port=80/tcp --permanent

**重启防火墙**

firewall-cmd –reload
