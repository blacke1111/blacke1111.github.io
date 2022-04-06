---
title: docker进阶
date: 2022-02-23 22:48:20
categories: Docker  
---

# 容器数据卷

 什么是容器数据卷

docker的理念回顾

将应用和环境打包成一个镜像！

容器的一个数据共享数据！Docker容器中产生的数据，同步到本地

这就是卷计数！目录的挂在，将我们容器内的目录，挂载到linux上面！

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220224182344.png)

**总结一句话，容器的持久化和同步操作！容器间也是可以数据共享的！**

> 方式一：直接使用命令来挂载 -v

```shell
#docker run -it -v 主机目录：容器内端口
docker run -it -v /home/ceshi:/home centos /bin/bash
```



docker inspect 容器id

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220224183715.png)

**source 主机内地址**

**Destination docker容器内的地址**

在主机上新建一个 test.java 自动同步到容器内

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220224184119.png)

**停止容器，再修改主机，容器启动后一样可以同步**

## **实战安装mysql**

```shell
#远程连接 mysql
mysql -h 192.168.218.129 -uroot -p -P 3306
#启动docker mysql
docker run -d -p 3310:3306 -v /home/mysql/conf:/etc/mysql/conf.d -v /home/mysql/data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=123456 --name mysql01 mysql

不知道为什么有bug，用本机的navicat连接不上docker的mysql  重启下虚拟机就解决了
```

假设我们将容器删除

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220224193504.png)

发现哦我们挂载到本地的数据卷依旧没有丢失，这就实现了容器数据持久化

## 具名和匿名挂载

```shell
#匿名挂载
docker run -d -P --name nginx01 -v /etc/nginx nginx  #这里nginx01是指容器外的名字

#查看所有卷情况
#docker volume ls
[root@hadoop01 data]# docker volume ls
DRIVER    VOLUME NAME
local     dd8860bc850ff056fbba682c113a8226bd2291b840d1deac21829845060f08c2

#具名挂载  -v 卷名：容器内路径
[root@hadoop01 data]# docker run -d --name nginx02 -v juming-nginx:/etc/nginx nginx
ee3232b4b1beb004c8275a59fe2af4e11aadbe8f0737e502ca7a5519d068223e
[root@hadoop01 data]# docker volume ls
DRIVER    VOLUME NAME
local     dd8860bc850ff056fbba682c113a8226bd2291b840d1deac21829845060f08c2
local     juming-nginx

#查看一下卷
[root@hadoop01 home]# docker volume inspect juming-nginx
[
    {
        "CreatedAt": "2022-02-24T19:45:05+08:00",
        "Driver": "local",
        "Labels": null,
        "Mountpoint": "/var/lib/docker/volumes/juming-nginx/_data",
        "Name": "juming-nginx",
        "Options": null,
        "Scope": "local"
    }
]
```

所有的docker容器内的卷，没有指定目录的情况下都是在 **"/var/lib/docker/volumes/xxxxxx/_data"**

小结：

```shell
#如何确定是具名挂载还是匿名挂载，还是指定路径挂载
-v 容器内路径 #匿名挂载
-v 卷名:容器内路径
-v /宿主机路径:容器内路径 #指定路径挂载
```



拓展：

```shell
#通过 -v 容器内路径:ro rw改变读写权限
ro readonly #只读
rw readwrite #可读可写

 docker run -d --name nginx02 -v juming-nginx:/etc/nginx:ro nginx
 
 #ro 只要看到ro就说明这个路径只能能通过宿主机来操作，容器内是无法操作的
```



## 初识DockerFile

Dockerfile就是用来构建镜像的构造文件！命令脚本！先体验下

```shell
#编写脚本文件
[root@hadoop01 docker-test-volume]# pwd
/home/docker-test-volume
[root@hadoop01 docker-test-volume]# vim dockerfile1
[root@hadoop01 docker-test-volume]# cat dockerfile1 
[root@hadoop01 docker-test-volume]# cat dockerfile1 
FROM centos

VOLUME ["volume01","volume02"]

CMD echo "-----end-----"

CMD /bin/bash
#注意最后有一个点
[root@hadoop01 docker-test-volume]# docker build -f dockerfile1  -t zhangcentos:1.0 .
Sending build context to Docker daemon  2.048kB
Step 1/4 : FROM centos
 ---> 5d0da3dc9764
Step 2/4 : VOLUME ["volume01","volume02"]
 ---> Running in a03689345bdd
Removing intermediate container a03689345bdd
 ---> 094ba8d8ac47
Step 3/4 : CMD echo "-----end-----"
 ---> Running in 3923319b0cc7
Removing intermediate container 3923319b0cc7
 ---> 9873922bfc87
Step 4/4 : CMD /bin/bash
 ---> Running in 30857643ba82
Removing intermediate container 30857643ba82
 ---> 3c1c95637190
Successfully built 3c1c95637190
Successfully tagged zhangcentos:1.0
[root@hadoop01 docker-test-volume]# docker images
REPOSITORY            TAG       IMAGE ID       CREATED              SIZE
zhangcentos           1.0       3c1c95637190   About a minute ago   231MB
tomcat02              1.0       4e4fd43e89dd   22 hours ago         684MB
mysql                 latest    17b062d639f4   6 days ago           519MB
tomcat                9.0       1fd0bf15d6bd   12 days ago          680MB
nginx                 latest    c316d5a335a5   4 weeks ago          142MB
centos                latest    5d0da3dc9764   5 months ago         231MB
portainer/portainer   latest    580c0e4e98b0   11 months ago        79.1MB
[root@hadoop01 docker-test-volume]# docker run -it zhangcentos /bin/bash
Unable to find image 'zhangcentos:latest' locally
#这里通过IMAGE ID 启动镜像
[root@hadoop01 docker-test-volume]# docker run -it 3c1c95637190 /bin/bash
[root@9c2631a9266a /]# ls -l
total 0
lrwxrwxrwx.   1 root root   7 Nov  3  2020 bin -> usr/bin
drwxr-xr-x.   5 root root 360 Feb 24 12:58 dev
drwxr-xr-x.   1 root root  66 Feb 24 12:58 etc
drwxr-xr-x.   2 root root   6 Nov  3  2020 home
lrwxrwxrwx.   1 root root   7 Nov  3  2020 lib -> usr/lib
lrwxrwxrwx.   1 root root   9 Nov  3  2020 lib64 -> usr/lib64
drwx------.   2 root root   6 Sep 15 14:17 lost+found
drwxr-xr-x.   2 root root   6 Nov  3  2020 media
drwxr-xr-x.   2 root root   6 Nov  3  2020 mnt
drwxr-xr-x.   2 root root   6 Nov  3  2020 opt
dr-xr-xr-x. 240 root root   0 Feb 24 12:58 proc
dr-xr-x---.   2 root root 162 Sep 15 14:17 root
drwxr-xr-x.  11 root root 163 Sep 15 14:17 run
lrwxrwxrwx.   1 root root   8 Nov  3  2020 sbin -> usr/sbin
drwxr-xr-x.   2 root root   6 Nov  3  2020 srv
dr-xr-xr-x.  13 root root   0 Feb 24 11:27 sys
drwxrwxrwt.   7 root root 171 Sep 15 14:17 tmp
drwxr-xr-x.  12 root root 144 Sep 15 14:17 usr
drwxr-xr-x.  20 root root 262 Sep 15 14:17 var
drwxr-xr-x.   2 root root   6 Feb 24 12:58 volume01
drwxr-xr-x.   2 root root   6 Feb 24 12:58 volume02


```

查看我们创建的镜像 的卷挂载的路径 docker inspect 容器id

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220224210041.png)

测试以下刚才的文件是否同步出去了

假设构建镜像的时候没有挂载，要手动镜像挂载 -v 卷名：容器内路径

# 数据卷挂载

多个mysql同步数据

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220224210214.png)

```shell
#启动三个容器 使用我们之气自己镜像的 zhangcentos
```

**docker01：**

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220224210935.png)

**docker02：**

```shell
#以docker01为父容器启动
docker run -d -it  --name docker02 --volumes-from docker01  zhangcentos:1.0  /bin/bash

```

docker01在colume中创建了docker01，可以看到在docker02（截图右边）中也可以看到

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220224211732.png)



**docker03**

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220224212138.png)



我通过查看docker inspect 信息看到docker01 docker02 docker03 的数据卷使用了相同的目录

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220224212949.png)

所以这样就实现了数据的共享



# DockerFile

 是用来构建docker镜像的文件！ 命令参数脚本！

**构建步骤：**

1.编写一个dockerfile文件

2.docker build 构建成文一个镜像

3.docker run 运行镜像

4.docker push 发布镜像（dockerhub,阿里云镜像仓库！）

查看一下官方怎么做的！

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220224213828.png)

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220224213843.png)

基础知识点：

1.每个保留关键字（指令）都必须是大写字母

2。执行从上到下顺序执行

3.# 表示注释

4. 每一个指令都会创建提交一个新的镜像层，并提交！



![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220224214214.png)

dockerfile是面向开发的。

DockerFile：构建文件，定义了一切的步骤，源代码

Dockerimages: 通过DockerFile构建生成的镜像，最终发布和运行的产品

## DockerFile的指令：



```shell
FROM      #基础镜像，一切从这里开始构建
MAINTAINER #镜像是谁写的，姓名+邮箱
RUN 	#镜像构建的时候需要运行的命令
ADD		#步骤： tomcat镜像，添加内容，就好比添加一个压缩包
WORKDIR	#镜像的工作目录
VOLIUME #挂载的目录位置
EXPOSE	#暴露端口
CMD		#指定这个容器启动的时候要运行的命令，只有最后一个生效，可被替代 我们自己docker run 时加命令替代
ENTRYPOINT #指定这个容器启动的时候要运行的命令，可以追加命令
ONBUILD 	#当构建一个被继承DockerFIle这个时候就会运行ONBUILD 的指令。触发命令
COPY		#类似ADD，将文件拷贝到镜像
ENV			#构建的时候设置环境变量
```



![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220224214810.png)



## 实战测试

```shell
#编写一个自己的centos
[root@hadoop01 dockerfile]# vim mydockerfile-centos
[root@hadoop01 dockerfile]# cat mydockerfile-centos 
FROM centos:7
MAINTAINER zhang<13954307478@qq.com>
ENV MYPATH /usr/local
WORKDIR $MYPATH
RUN yum -y install vim
RUN yum -y install net-tools

EXPOSE 80
CMD echo $MYPATH
CMD echo "-----end----"
CMD /bin/bash
[root@hadoop01 dockerfile]# pwd
/home/dockerfile
[root@hadoop01 dockerfile]# 
# 2构建文件的镜像
docker build -f mydockerfile-centos -t mycentos:0.1 .
```

如果报错 ：Error: Failed to download metadata for repo 'AppStream'

Centos8不再维护，在2022年1月31日，CentOS团队从官方镜像中移除CentOS 8的所有包。
解决方式：dockerfile第一行FROM centos:7



```shell
docker history 可以查看镜像的历史
```

> CMD和ENTRYPONIT区别

```shell
CMD #指定这个容器启动的时候要运行的命令，只有最后一个会生效，可被替代
ENTRYPOINT #指定这个容器启动的时候要运行的命令，可以追加命令
```

测试CMD

```shell
[root@hadoop01 dockerfile]# vim dockerfile-cmd-test
[root@hadoop01 dockerfile]# cat dockerfile-cmd-test 
FROM centos
CMD ["ls","-a"]
[root@hadoop01 dockerfile]# docker build -f dockerfile-cmd-test -t cmdtest:1.0 .
Sending build context to Docker daemon  3.072kB
Step 1/2 : FROM centos
 ---> 5d0da3dc9764
Step 2/2 : CMD ["ls","-a"]
 ---> Running in 878e2dc278be
Removing intermediate container 878e2dc278be
 ---> 6aeaf1361fbe
Successfully built 6aeaf1361fbe
Successfully tagged cmdtest:1.0
[root@hadoop01 dockerfile]# docker run cmdtest:1.0
.
..
.dockerenv
bin
dev
etc
home
lib
lib64
lost+found
media
mnt
opt
proc
root
run
sbin
srv
sys
tmp
usr
var
#想追加一个命令 -l
[root@hadoop01 dockerfile]# docker run cmdtest:1.0 -l
docker: Error response from daemon: OCI runtime create failed: container_linux.go:380: starting container process caused: exec: "-l": executable file not found in $PATH: unknown.
ERRO[0004] error waiting for container: context canceled 
#出错 cmd的清理下 -l 替换了CMd["ls","-a"]命令，-l不是命令所以报错
```

测试 ENTRYPOINT

```shell
[root@hadoop01 dockerfile]# cat dockerfile-cmd-test 
FROM centos
ENTRYPOINT ["ls","-a"]
[root@hadoop01 dockerfile]# docker build -f dockerfile-cmd-test -t entorypoint-test .
Sending build context to Docker daemon  3.072kB
Step 1/2 : FROM centos
 ---> 5d0da3dc9764
Step 2/2 : ENTRYPOINT ["ls","-a"]
 ---> Running in 495b83c71774
Removing intermediate container 495b83c71774
 ---> b5c203ad0353
Successfully built b5c203ad0353
Successfully tagged entorypoint-test:latest
[root@hadoop01 dockerfile]# docker images
REPOSITORY            TAG       IMAGE ID       CREATED          SIZE
entorypoint-test      latest    b5c203ad0353   23 seconds ago   231MB
cmdtest               1.0       6aeaf1361fbe   13 minutes ago   231MB
<none>                <none>    e3e3abd1182b   19 minutes ago   231MB
zhangcentos           1.0       65fe23d38a98   2 hours ago      231MB
tomcat02              1.0       4e4fd43e89dd   24 hours ago     684MB
mysql                 latest    17b062d639f4   6 days ago       519MB
tomcat                9.0       1fd0bf15d6bd   12 days ago      680MB
nginx                 latest    c316d5a335a5   4 weeks ago      142MB
centos                latest    5d0da3dc9764   5 months ago     231MB
portainer/portainer   latest    580c0e4e98b0   11 months ago    79.1MB
[root@hadoop01 dockerfile]# docker run  entorypoint-test
.
..
.dockerenv
bin
dev
etc
home
lib
lib64
lost+found
media
mnt
opt
proc
root
run
sbin
srv
sys
tmp
usr
var
[root@hadoop01 dockerfile]# docker run  entorypoint-test -l
total 0
drwxr-xr-x.   1 root root   6 Feb 24 14:55 .
drwxr-xr-x.   1 root root   6 Feb 24 14:55 ..
-rwxr-xr-x.   1 root root   0 Feb 24 14:55 .dockerenv
lrwxrwxrwx.   1 root root   7 Nov  3  2020 bin -> usr/bin
drwxr-xr-x.   5 root root 340 Feb 24 14:55 dev
drwxr-xr-x.   1 root root  66 Feb 24 14:55 etc
drwxr-xr-x.   2 root root   6 Nov  3  2020 home
lrwxrwxrwx.   1 root root   7 Nov  3  2020 lib -> usr/lib
lrwxrwxrwx.   1 root root   9 Nov  3  2020 lib64 -> usr/lib64
drwx------.   2 root root   6 Sep 15 14:17 lost+found
drwxr-xr-x.   2 root root   6 Nov  3  2020 media
drwxr-xr-x.   2 root root   6 Nov  3  2020 mnt
drwxr-xr-x.   2 root root   6 Nov  3  2020 opt
dr-xr-xr-x. 258 root root   0 Feb 24 14:55 proc
dr-xr-x---.   2 root root 162 Sep 15 14:17 root
drwxr-xr-x.  11 root root 163 Sep 15 14:17 run
lrwxrwxrwx.   1 root root   8 Nov  3  2020 sbin -> usr/sbin
drwxr-xr-x.   2 root root   6 Nov  3  2020 srv
dr-xr-xr-x.  13 root root   0 Feb 24 11:27 sys
drwxrwxrwt.   7 root root 171 Sep 15 14:17 tmp
drwxr-xr-x.  12 root root 144 Sep 15 14:17 usr
drwxr-xr-x.  20 root root 262 Sep 15 14:17 var

```

## 实战tomcat：

我在我的/usr/local/software 下添加了tomcat和jdk8的压缩包

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220224233810.png)

2，编写dockerfile文件，官方命名 mor Dockerfile ，build会自动寻找这个文件 ，就不要要-f指定了

```shell
FROM centos:7
MAINTAINER zhang<1395430748@qq.com>

COPY readme.txt /usr/local/readme.txt
ADD openjdk-8u41-b04-linux-x64-14_jan_2020.tar.gz /usr/local
ADD apache-tomcat-8.5.63.tar.gz /usr/local
RUN yum -y install vim
ENV MYPATH /usr/local
WORKDIR $MYPATH
ENV JAVA_HOME /usr/local/java-se-8u41-ri
ENV CLASSPATH $JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
ENV CATELINA_HOME /usr/local/apache-tomcat-8.5.63
ENV CATALINA_BASH /usr/local/apache-tomcat-8.5.63
ENV PATH $PATH:$JAVA_HOME:$CATALINA_HOME/lib:$CATALINA_HOME/bin

EXPOSE 8080
CMD /usr/local/apache-tomcat-8.5.63/bin/startup.sh && tail -F /usr/local/apache-tomcat-8.5.63/logs/catalina.out
```

3.构建镜像

```shell
docker build -t diytomcat .
#运行镜像
docker run -d -p 9090:8080 --name zhangtomcat -v /home/tomcat/test:/usr/local/apache-tomcat-8.5.63/webapps/test -v /home/tomcat/tomcatlogs:/usr/local/apache-tomcat-8.5.63/logs diytomcat

docker exec -it 0029585f80d9 /bin/bash
```

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220225011525.png)

web.xml

```java
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
         version="4.0">
    
</web-app>
```

inddex.jsp:

```java
<%@ page language="java" contentType="text/html; charset=UTF-8"
    pageEncoding="UTF-8"%>
<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
<title>菜鸟教程(runoob.com)</title>
</head>
<body>
Hello World!<br/>
<%
System.out.println("hello ,world");
%>
</body>
</html>
```

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220225011553.png)

## 发布镜像：

## 

> dockerhub

1，地址：https://hub.docker.com/ 注册自己的帐号

2.登录账号

3.在我们的服务器上提交

```shell
[root@hadoop01 ~]# docker login --help

Usage:  docker login [OPTIONS] [SERVER]

Log in to a Docker registry.
If no server is specified, the default is defined by the daemon.

Options:
  -p, --password string   Password
      --password-stdin    Take the password from stdin
  -u, --username string   Username
[root@hadoop01 ~]# docker login -u 1395430748
Password: 
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```

4.登陆完毕就可以提交 命令：docker push

```shell
#push自己的镜像服务器上
[root@hadoop01 ~]# docker push diytomcat
Using default tag: latest
The push refers to repository [docker.io/library/diytomcat]
d7850627f70d: Preparing 
6af465e86edd: Preparing 
febfb7d17978: Preparing 
cb88cb10a889: Preparing 
174f56854903: Preparing 
denied: requested access to the resource is denied #拒绝

#添加标签
docker tag 要提交的容器id 1395430748/diytomcat

```

## 小结：

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220226172002.png)

# Docker网络

## 理解Docker0



linux查看网络：

```shell
ip addr
```

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220226172722.png)

lo:本机回环地址

ens33 linux网卡

docker 虚拟网卡

```shell
问题： docker是如何处理容器网络访问的？

```

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220226173029.png)

```shell
#运行tomcat
docker run -d -P --name tomcat01 tomcat

#在容器内查看容器内部网络地址
docker inspect 容器名称或 id
```

问题解决：

tomcat镜像不带ip addr和ping命令：

在容器内执行：

```shell
root@c9478f4e83dd:/usr/local/tomcat# apt update
root@c9478f4e83dd:/usr/local/tomcat# apt install -y iproute2
root@c9478f4e83dd:/usr/local/tomcat# apt install iputils-ping
```

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220226174357.png)

```she
可以看到我容器和我的宿主机在一个网段，所以可以ping通
```

![image-20220226180101543](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20220226180101543.png)

网卡是一对一对的

虚拟机的78序号网卡 对应着容器内 79号网卡

```shell
#这种计数就是 evth-pair 他们都是成对出现的，一段留着协议，一段彼此相连
#正因为有这个特性，eveh-pair 充当一个桥梁，连接各种虚拟网络设备的
```

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220226181745.png)

Docker使用的是linux的桥接技术

只要容器删除，对应的网桥也会消失

> 思考一个场景，我们编写一个微服务，database url=ip 项目部重启，数据库ip改变 ，我们项目不需要更改id  这就好比我们学习springcloud的时候 消费端调用服务端，ip地址不写死，而是他的服务名称，再去注册中兴中找到真实的服务者ip，现在我们希望docker可以处理这个问题，通过名字访问容器？

首先

```shell
[root@hadoop01 ~]# docker exec -it 7e96865e7db3 ping tomcat01
ping: tomcat01: Name or service not known
找不带tomcat01
```

--link

```shell

[root@hadoop01 ~]# docker run -d -P --name tomcat02 --link tomcat01 mytomcat:1.0
cb86c2617d19ee4a5d3298a53e74c8f3bc3061de143204eb98d4b5a4cbbacc17
[root@hadoop01 ~]# docker ps
CONTAINER ID   IMAGE          COMMAND             CREATED             STATUS          PORTS                                         NAMES
cb86c2617d19   mytomcat:1.0   "catalina.sh run"   9 seconds ago       Up 8 seconds    0.0.0.0:49154->8080/tcp, :::49154->8080/tcp   tomcat02
7e96865e7db3   tomcat         "catalina.sh run"   36 minutes ago      Up 31 minutes   0.0.0.0:3344->8080/tcp, :::3344->8080/tcp     charming_dewdney
c9a1c0930ec8   tomcat         "catalina.sh run"   About an hour ago   Up 31 minutes   0.0.0.0:49153->8080/tcp, :::49153->8080/tcp   tomcat01
[root@hadoop01 ~]# docker exec -it tomcat02 ping tomcat01
PING tomcat01 (172.17.0.2) 56(84) bytes of data.
64 bytes from tomcat01 (172.17.0.2): icmp_seq=1 ttl=64 time=0.079 ms
64 bytes from tomcat01 (172.17.0.2): icmp_seq=2 ttl=64 time=0.076 ms
64 bytes from tomcat01 (172.17.0.2): icmp_seq=3 ttl=64 time=0.072 ms

#但是我们反向ping 是不可以的 因为没有绑定

```

在tomcat02中可以看到links信息：

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220226190009.png)

etc/host文件中也可以看到： 当我们访问tomcat01时就是访问的ip 172.17.0.2 而这个ip就是我们的tomcat01的ip地址

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220226190316.png)

而tomcat01中：

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220226190112.png)

查看所有docker网络

```shell
[root@hadoop01 ~]# docker network --help
Usage:  docker network COMMAND
Manage networks
Commands:
  connect     Connect a container to a network
  create      Create a network
  disconnect  Disconnect a container from a network
  inspect     Display detailed information on one or more networks
  ls          List networks
  prune       Remove all unused networks
  rm          Remove one or more networks
Run 'docker network COMMAND --help' for more information on a command.

[root@hadoop01 ~]# docker network ls
NETWORK ID     NAME      DRIVER    SCOPE
4bdef296ac71   bridge    bridge    local
fde3b2c9dac3   host      host      local
46c769d1a278   none      null      local

4bdef296ac71 这个是我们的docker0
[root@hadoop01 ~]# docker inspect 4bdef296ac71
[
    {
        "Name": "bridge",
        "Id": "4bdef296ac719d1435c65b58833b7224eff8e1491545a995aa5158cfcf57f8f3",
        "Created": "2022-02-26T18:13:06.121965705+08:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16",
                    "Gateway": "172.17.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "7e96865e7db3a8c8fb061580a91a51a7dbb7b7c521dbfa67624c1fc73c73a2b1": {
                "Name": "charming_dewdney",
                "EndpointID": "9da762ee45e30dad05680029a4154bd68b538143a56d27b61c35d2d8260ca2c4",
                "MacAddress": "02:42:ac:11:00:03",
                "IPv4Address": "172.17.0.3/16",
                "IPv6Address": ""
            },
            "c9a1c0930ec836650adad4ca886328d87c53af03874dd49900f6a315b01013a8": {
                "Name": "tomcat01",
                "EndpointID": "134e196c65420a8e09bdbb911d5001cfa280d8ab448cf0b4e0797820c6f9a26b",
                "MacAddress": "02:42:ac:11:00:02",
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            },
            "cb86c2617d19ee4a5d3298a53e74c8f3bc3061de143204eb98d4b5a4cbbacc17": {
                "Name": "tomcat02",
                "EndpointID": "7307090942d607b710aee5be570ce62ef5aaf92922963257d61d080e74640b28",
                "MacAddress": "02:42:ac:11:00:04",
                "IPv4Address": "172.17.0.4/16",
                "IPv6Address": ""
            }
        },
        "Options": {
            "com.docker.network.bridge.default_bridge": "true",
            "com.docker.network.bridge.enable_icc": "true",
            "com.docker.network.bridge.enable_ip_masquerade": "true",
            "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
            "com.docker.network.bridge.name": "docker0",
            "com.docker.network.driver.mtu": "1500"
        },
        "Labels": {}
    }
]

```

网桥：

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220226185030.png)

docker0问题：它不支持我们容器名访问！

## 自定义网络：

> 查看所有docker网络

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220226191235.png)

网络模式：

bridge：桥接模式 

none：不配置网络

host：和宿主机共享网络

测试：

```shell
# 我们直接启动的命令 --net brigde 而这个是我们的docker0
docker run -d -P--name tomcat01 --net brigde tomcat

#docker0特点： 默认，域名不能访问，--link可以打通 但是不建议

--driver bridge  桥接模式的网络
--subnet 192.168.0.0/16 子网地址在这个掩码范围内
--gateway 网关
#创建自定义网络
docker network create --driver bridge --subnet 192.168.0.0/16 --gateway 192.168.0.1 mynet
```

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220226192113.png)

```shell
#这里mytomcat 这里mytomcat镜像我是添加了ping和ip addr命令
[root@hadoop01 ~]# docker run -d -P --name tomcat-net-01 --net mynet mytomcat:1.0
1430a1eb9adf971e90e5779e723b4a400737d487f52349a671b0fe0b74f3c8a9
[root@hadoop01 ~]# docker run -d -P --name tomcat-net-02 --net mynet mytomcat:1.0
0df792d4e69507af37b2eafaa94ff2deecbea63ddd9531195669625907e7879e
#查看mynet
[root@hadoop01 ~]# docker inspect  4a767f605981
[
    {
        "Name": "mynet",
        "Id": "4a767f6059811888557037ba1d5e8b0803c881ef9bab438e12d704524661c4ca",
        "Created": "2022-02-26T19:18:50.708325835+08:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "192.168.0.0/16",
                    "Gateway": "192.168.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "0df792d4e69507af37b2eafaa94ff2deecbea63ddd9531195669625907e7879e": {
                "Name": "tomcat-net-02",
                "EndpointID": "90c002b405485825844210e172376b7ae028865d015ac25dae65a3bb6e2ea66e",
                "MacAddress": "02:42:c0:a8:00:03",
                "IPv4Address": "192.168.0.3/16",
                "IPv6Address": ""
            },
            "1430a1eb9adf971e90e5779e723b4a400737d487f52349a671b0fe0b74f3c8a9": {
                "Name": "tomcat-net-01",
                "EndpointID": "45b492f793dfc7bda9df2d55e41c6e87d4b0416c3197ebe1d3b96209ea653557",
                "MacAddress": "02:42:c0:a8:00:02",
                "IPv4Address": "192.168.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]


```

**测试 通过容器名ping**

不使用--link ，也可以通过容器ping通

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220226192539.png)

## 网络连通：



使用docker network connect 【options】 网络名称 容器名称

```shell
[root@hadoop01 ~]# docker network connect --help

Usage:  docker network connect [OPTIONS] NETWORK CONTAINER

Connect a container to a network

Options:
      --alias strings           Add network-scoped alias for the container
      --driver-opt strings      driver options for the network
      --ip string               IPv4 address (e.g., 172.30.100.104)
      --ip6 string              IPv6 address (e.g., 2001:db8::33)
      --link list               Add link to another container
      --link-local-ip strings   Add a link-local address for the container
      
#测试 tomcat01 - tomcat-net-01
[root@hadoop01 ~]# docker network connect mynet tomcat01
[root@hadoop01 ~]# docker exec -it tomcat01 ping tomcat-net-01
PING tomcat-net-01 (192.168.0.2) 56(84) bytes of data.
64 bytes from tomcat-net-01.mynet (192.168.0.2): icmp_seq=1 ttl=64 time=0.128 ms
64 bytes from tomcat-net-01.mynet (192.168.0.2): icmp_seq=2 ttl=64 time=0.044 ms
64 bytes from tomcat-net-01.mynet (192.168.0.2): icmp_seq=3 ttl=64 time=0.052 ms
64 bytes from tomcat-net-01.mynet (192.168.0.2): icmp_seq=4 ttl=64 time=0.046 ms

```

我们再来查看mynet网络

```shell
docker inspect mynet
```

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220226195652.png)

可以看到他把我们的tomcat01在mynet网络上注册了一个ip地址

# 实战redis集群：

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220226200110.png)

shell脚本！

```shell
#创建redis网络
docker network create redis --subnet 172.38.0.0/16

#shell 脚本
for port in $(seq 1 6); \
do \
mkdir -p /home/mydata/redis/node-${port}/conf
touch /home/mydata/redis/node-${port}/conf/redis.conf
cat << EOF >/home/mydata/redis/node-${port}/conf/redis.conf
port 6379 
bind 0.0.0.0
cluster-enabled yes 
cluster-config-file nodes.conf
cluster-node-timeout 5000
cluster-announce-ip 172.38.0.1${port}
cluster-announce-port 6379
cluster-announce-bus-port 16379
appendonly yes
EOF
done

#启动redis --ip指定在redis网络 中的ip
#1
 docker run -p 6371:6379 -p 16371:16379 --name redis-1 \
 -v /home/mydata/redis/node-1/data:/data \
 -v /home/mydata/redis/node-1/conf/redis.conf:/etc/redis/redis.conf \
 -d --net redis --ip 172.38.0.11 redis:5.0.9-alpine3.11 redis-server /etc/redis/redis.conf
 #2
  docker run -p 6372:6379 -p 16372:16379 --name redis-2 \
 -v /home/mydata/redis/node-2/data:/data \
 -v /home/mydata/redis/node-2/conf/redis.conf:/etc/redis/redis.conf \
 -d --net redis --ip 172.38.0.12 redis:5.0.9-alpine3.11 redis-server /etc/redis/redis.conf
 #3
  docker run -p 6373:6379 -p 16373:16379 --name redis-3 \
 -v /home/mydata/redis/node-3/data:/data \
 -v /home/mydata/redis/node-3/conf/redis.conf:/etc/redis/redis.conf \
 -d --net redis --ip 172.38.0.13 redis:5.0.9-alpine3.11 redis-server /etc/redis/redis.conf
 #4
  docker run -p 6374:6379 -p 16374:16379 --name redis-4 \
 -v /home/mydata/redis/node-4/data:/data \
 -v /home/mydata/redis/node-4/conf/redis.conf:/etc/redis/redis.conf \
 -d --net redis --ip 172.38.0.14 redis:5.0.9-alpine3.11 redis-server /etc/redis/redis.conf
 #5
  docker run -p 6375:6379 -p 16375:16379 --name redis-5 \
 -v /home/mydata/redis/node-5/data:/data \
 -v /home/mydata/redis/node-5/conf/redis.conf:/etc/redis/redis.conf \
 -d --net redis --ip 172.38.0.15 redis:5.0.9-alpine3.11 redis-server /etc/redis/redis.conf
 #6
  docker run -p 6376:6379 -p 16376:16379 --name redis-6 \
 -v /home/mydata/redis/node-6/data:/data \
 -v /home/mydata/redis/node-6/conf/redis.conf:/etc/redis/redis.conf \
 -d --net redis --ip 172.38.0.16 redis:5.0.9-alpine3.11 redis-server /etc/redis/redis.conf
 
 
 #进去redis
  docker exec -it  redis-1 /bin/sh
 # 进去redis-1后，创建集群
redis-cli --cluster create 172.38.0.11:6379 172.38.0.12:6379 172.38.0.13:6379 172.38.0.14:6379 172.38.0.15:6379 172.38.0.16:6379 --cluster-replicas 1

>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 172.38.0.15:6379 to 172.38.0.11:6379
Adding replica 172.38.0.16:6379 to 172.38.0.12:6379
Adding replica 172.38.0.14:6379 to 172.38.0.13:6379
M: 0051012c2c4bd5ab9b463147e09ded322a1d0fe4 172.38.0.11:6379
   slots:[0-5460] (5461 slots) master
M: 8effc3f637bfdefbd7d796bc452c974abc3af036 172.38.0.12:6379
   slots:[5461-10922] (5462 slots) master
M: 1bd71eeda965933e2d2cdb24964407847583abcf 172.38.0.13:6379
   slots:[10923-16383] (5461 slots) master
S: cb5836b251d20352a284b8a85fcbd1cf36e4789b 172.38.0.14:6379
   replicates 1bd71eeda965933e2d2cdb24964407847583abcf
S: bda88d5c20ae9c7c34511dd07ef6b10c65fdb6da 172.38.0.15:6379
   replicates 0051012c2c4bd5ab9b463147e09ded322a1d0fe4
S: 4c0020662a4ad69e9463efa76d4b91e4a4de8190 172.38.0.16:6379
   replicates 8effc3f637bfdefbd7d796bc452c974abc3af036
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join
..
>>> Performing Cluster Check (using node 172.38.0.11:6379)
M: 0051012c2c4bd5ab9b463147e09ded322a1d0fe4 172.38.0.11:6379
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
S: 4c0020662a4ad69e9463efa76d4b91e4a4de8190 172.38.0.16:6379
   slots: (0 slots) slave
   replicates 8effc3f637bfdefbd7d796bc452c974abc3af036
S: cb5836b251d20352a284b8a85fcbd1cf36e4789b 172.38.0.14:6379
   slots: (0 slots) slave
   replicates 1bd71eeda965933e2d2cdb24964407847583abcf
M: 8effc3f637bfdefbd7d796bc452c974abc3af036 172.38.0.12:6379
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
M: 1bd71eeda965933e2d2cdb24964407847583abcf 172.38.0.13:6379
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
S: bda88d5c20ae9c7c34511dd07ef6b10c65fdb6da 172.38.0.15:6379
   slots: (0 slots) slave
   replicates 0051012c2c4bd5ab9b463147e09ded322a1d0fe4
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.

#连接redis集群 -c是集群连接
redis-cli -c

#可以看到是三主三从
127.0.0.1:6379> cluster nodes
4c0020662a4ad69e9463efa76d4b91e4a4de8190 172.38.0.16:6379@16379 slave 8effc3f637bfdefbd7d796bc452c974abc3af036 0 1645878439512 6 connected
0051012c2c4bd5ab9b463147e09ded322a1d0fe4 172.38.0.11:6379@16379 myself,master - 0 1645878437000 1 connected 0-5460
cb5836b251d20352a284b8a85fcbd1cf36e4789b 172.38.0.14:6379@16379 slave 1bd71eeda965933e2d2cdb24964407847583abcf 0 1645878438000 4 connected
8effc3f637bfdefbd7d796bc452c974abc3af036 172.38.0.12:6379@16379 master - 0 1645878439000 2 connected 5461-10922
1bd71eeda965933e2d2cdb24964407847583abcf 172.38.0.13:6379@16379 master - 0 1645878438505 3 connected 10923-16383
bda88d5c20ae9c7c34511dd07ef6b10c65fdb6da 172.38.0.15:6379@16379 slave 0051012c2c4bd5ab9b463147e09ded322a1d0fe4 0 1645878439513 5 connected
#设置a 值为b
127.0.0.1:6379> set a b
-> Redirected to slot [15495] located at 172.38.0.13:6379
OK

#关闭redis-3
[root@hadoop01 ~]# docker stop redis-3

#再去获取 a get a
127.0.0.1:6379> get a
-> Redirected to slot [15495] located at 172.38.0.14:6379
"b"
```

可以看到ip 14的作为了13的主机

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220226203629.png)

# 实战springBoot项目：

idea编写一个简单的springboot项目：

就简单的写了一个controller

```java
@RestController
public class HelloController {

    @RequestMapping("/hello")
    public String hello(){
        return "hello,dockerTest!!!!";
    }
}
```

使用maven打包：

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220226204851.png)

安装idea插件可以连接docker远程仓库

![](
https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220226205014.png)



编写Dockerfile文件

```dockerfile
FROM java:8
COPY *.jar /app.jar
CMD ["--server.port=8088"]
EXPOSE 8088
ENTRYPOINT ["java","-jar","app.jar"]
```

把jar包和Dockerfile文件上传到linxu上

```shell
#上传之后查看 是否上传成功
[root@hadoop01 idea]# ls
demo-0.0.1-SNAPSHOT.jar  Dockerfile
#生成镜像
docker build -t zhang666 .

#运行镜像
docker run  -d -P --name springboot-test zhang666
#访问
[root@hadoop01 idea]# docker ps
CONTAINER ID   IMAGE      COMMAND                  CREATED         STATUS         PORTS                                         NAMES
074a6bf6f7d8   zhang666   "java -jar app.jar -…"   5 seconds ago   Up 4 seconds   0.0.0.0:49160->8088/tcp, :::49160->8088/tcp   springboot-test
[root@hadoop01 idea]# curl 192.168.32.3:49160/hello
hello,dockerTest!!!!

#电脑上浏览器访问不了 执行以下命令
iptables -P FORWARD ACCEPT
systemctl start docker
#end
```
