---
title: Docker入门
date: 2022-02-22 19:56:36
categories: Docker
---

# Docker的组成

镜像（image）： docker镜像就好比一个模板， 可以通过这个模板来创建容器服务，tomcat镜像===》run===》tomcat1容器（提供服务器），通过这个镜像可以创建多个容器

容器（container）：

Docker利用容器技术，独立运行一个或者一组应用，通过镜像来创建，启动，停止，删除，基本命令

目前就可以把这个容器理解为就是一个简易的linux系统

仓库（repository）：

仓库就是来存放镜像的的地方，分为私有和公有

# Docker安装

## 环境准备

linux服务器，我使用的vm虚拟机，centos7

### 环境查看

```shell
[root@hadoop01 ~]# uname -r
3.10.0-957.el7.x86_64
```

```shell
[root@hadoop01 ~]# cat /etc/os-release
NAME="CentOS Linux"
VERSION="7 (Core)"
ID="centos"
ID_LIKE="rhel fedora"
VERSION_ID="7"
PRETTY_NAME="CentOS Linux 7 (Core)"
ANSI_COLOR="0;31"
CPE_NAME="cpe:/o:centos:centos:7"
HOME_URL="https://www.centos.org/"
BUG_REPORT_URL="https://bugs.centos.org/"
CENTOS_MANTISBT_PROJECT="CentOS-7"
CENTOS_MANTISBT_PROJECT_VERSION="7"
REDHAT_SUPPORT_PRODUCT="centos"
REDHAT_SUPPORT_PRODUCT_VERSION="7"

docker在我们本机的目录：
/var/lib/docker

```

### 安装

```shell
#1.卸载旧版本
yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
                  
#2。需要的安装包
yum install -y yum-utils
#3.设置镜像的仓库 
yum-config-manager \
    --add-repo \
    http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo #阿里云
    https://download.docker.com/linux/centos/docker-ce.repo #默认国外
    
 #更新软件包的索引
 yum makecache fast
#4.安装docker引擎 docker-ce 社区 ee企业
yum install docker-ce docker-ce-cli containerd.io

#5 启动docker
systemctl start docker

#6.查看是否安装成功
 docker version 


```

```shell
 #7.测试hello world
 docker run hello-world
```

![]( https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220222212425.png)

```shell
#8.查看下载的镜像
```



![]( https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220222212322.png)

```shell
#卸载docker
#1卸载依赖
yum remove docker-ce docker-ce-cli containerd.io
#2删除资源
rm -rf /var/lib/docker
rm -rf /var/lib/containerd
```



docker怎么工作的？

Docker是一个 Client-Server结构的系统 Docker的守护进程运行在主机上。通过Socket从客户端访问

DockerServer接受到Docker-client的指令，就会执行这个命令

![]( https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220222213525.png)



Docker为什么比vm快？

1。Docker有着比虚拟机更少的抽象层

2.docker利用的是宿主机的内核 ， vm需要Guest OS

![]( https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220222213659.png)



# Docker的常用命令：

## 帮助命令

```shell
docker version   #显示docker的版本信息
docker info      #显示docker的系统信息，包括镜像和容器的数量
docker 命令 --help 
```



官方文档地址:https://docs.docker.com/reference/

## 镜像命令：

docker images 查看所有本地的主机上的镜像

```shell
docker images
REPOSITORY    TAG       IMAGE ID       CREATED        SIZE   
hello-world   latest    feb5d9fea6a5   5 months ago   13.3kB
# 解释
REPOSITORY 镜像的仓库源
TAG      镜像的标签
IMAGE ID 镜像的id
CREATED 镜像的创建时间
SIZE   镜像的大小

#可选项
 -a, --all             #列出所有镜像
 -q, --quiet           #只显示镜像id
  -f, --filter filter   #
      --format string   Pretty-print images using a Go template
      --no-trunc        Don't truncate output
  

```



**docker search 搜索镜像**

```shell
[root@hadoop01 /]# docker search mysql
NAME                             DESCRIPTION                                     STARS     OFFICIAL   AUTOMATED
mysql/mysql-server               Optimized MySQL Server Docker images. Create…   906                  [OK]
mysql/mysql-cluster              Experimental MySQL Cluster Docker images. Cr…   92                   
centos/mysql-57-centos7          MySQL 5.7 SQL database server                   92    

#可选项 ，通过搜藏来过滤
--filter=STARS=3000 #搜索出来的镜像就是STARS大于3000的
```

**docker pull 下载镜像**

```shell
#下载镜像 docker docker pull 镜像名[:tag]
[root@hadoop01 /]# docker search mysql --filter=STARS=3000
NAME      DESCRIPTION   STARS     OFFICIAL   AUTOMATED
[root@hadoop01 /]# docker pull mysql
Using default tag: latest  //如果不写tag，默认就是latest
latest: Pulling from library/mysql
6552179c3509: Pull complete   //分层下载，docker image的核心 联合文件系统
d69aa66e4482: Pull complete 
3b19465b002b: Pull complete 
7b0d0cfe99a1: Pull complete 
9ccd5a5c8987: Pull complete 
2dab00d7d232: Pull complete 
5d726bac08ea: Pull complete 
11bb049c7b94: Pull complete 
7fcdd679c458: Pull complete 
11585aaf4aad: Pull complete 
5b5dc265cb1d: Pull complete 
fd400d64ffec: Pull complete 
Digest: sha256:e3358f55ea2b0cd432685d7e3c79a33a85c7a359b35fa87fc4993514b9573446
Status: Downloaded newer image for mysql:latest
docker.io/library/mysql:latest
#等价于
docker pull mysql
docker pull docker.io/library/mysql:latest

版本一定要对应
```

![]( https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220222221448.png)



**docker rmi 删除镜像**

**i代表image**

```shell
docker rmi -f 镜像id||镜像名称

#递归删除
docker rmi -f $(docker images -aq)
```

## 容器命令

有了镜像才可以创建容器，linux 下载一个centos镜像来学习

```shell
socker pull centos
```

**新建容器并启动**

```shell
docker run [可选参数] image

#参数说明
--name="Name" 容器名字 tomcat01 tomcat02 来区分容器
-d            后台方式运行
-it           使用交互方式运行，进入容器查看内容
-p            指定容器端口 -p 8080:8080
	-p ip:主机端口：容器端口
	-p 主机端口：容器端口
	-p 容器端口
-P            随机指定端口       


#测试 ,
[root@hadoop01 /]# docker run -it centos /bin/bash
[root@bda81521c40c /]# 

#小型的容器
[root@bda81521c40c /]# ls   #查看容器内的centos，基础版本，很多命令都是不完善的
bin  etc   lib	  lost+found  mnt  proc  run   srv  tmp  var
dev  home  lib64  media       opt  root  sbin  sys  usr

```

小提示： 使用容器名字启动 如果不知pull的最新版一定要带TAG去执行 不然会去下载最新版

**退出容器**

```shell
exit #直接容器退出并停止
键盘上 按住 Ctrl+p+q #退出不停止
[root@hadoop01 ~]# docker run -it centos /bin/bash
[root@8c2d40d8b7f8 /]# [root@hadoop01 ~]# docker ps
CONTAINER ID   IMAGE     COMMAND       CREATED              STATUS              PORTS     NAMES
8c2d40d8b7f8   centos    "/bin/bash"   About a minute ago   Up About a minute             keen_mahavira

```



**列出所有的运行容器**

```shell
[root@hadoop01 ~]# docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES

[root@hadoop01 ~]# docker ps -a
CONTAINER ID   IMAGE         COMMAND       CREATED             STATUS                         PORTS     NAMES
bda81521c40c   centos        "/bin/bash"   5 minutes ago       Exited (0) 2 minutes ago                 amazing_wilbur
fdc5041e2e0b   hello-world   "/hello"      About an hour ago   Exited (0) About an hour ago             pensive_kilby
78fbde335780   hello-world   "/hello"      4 months ago        Exited (0) 4 months ago                  vigorous_merkle

-a #查看当前正在运行的容器+曾经运行过的
-n=? #流出最近创建的容器
-q #只显示容器的编号
```

**删除容器**

```shell
docker rm 容器id  #删除指定的容器，不能删除正在运行的 如果要强制删除必须加-f
docker rm -f $(docker ps -aq) #递归删除所有容器
docker ps -a -q|xargs docker rm #删除所有容器
```

**启动和停止容器的操作**

```shell
docker start 容器id       #启动容器
docker restart 容器id	    #重启容器
docker stop 容器id		#停止当前容器
docker kill 容器id		#强制停止当前容器

```

## 常用的其他命令

```shell
# 命令docker run -d 镜像名！
[root@hadoop01 ~]# docker run -d centos
# 问题docker ps 发现centos停止了
#常见的坑 docker，容器使用后运行，就必须要有一个前台进程，docker发现没有应用，就会自动停止
#nginx ，容器启动后，发现自己没有提供服务，就会立即停止，就只没有程序了
```

**查看日志**

```shell
docker logs -f -t --tail 容器，没有日志

#自己编写一段shell脚本
[root@hadoop01 ~]# docker run -d centos /bin/sh -c "while true;do echo zhang;sleep 1;done"
[root@hadoop01 ~]# docker ps
CONTAINER ID   IMAGE     COMMAND                  CREATED         STATUS         PORTS     NAMES
130b65023170   centos    "/bin/sh -c 'while t…"   3 seconds ago   Up 2 seconds             boring_mayer
#显示日志
-tf # 显示日志 f是带上时间戳
--tail number #显示日志的行数number
[root@hadoop01 ~]# docker logs -tf --tail 10 130b65023170


```

**查看容器中的进程进程**

```shell
[root@hadoop01 ~]# docker top 130b65023170  #查看容器中的进程
```

**查看镜像的元数据·**

```shell
docker inspect 容器id
```

**进入当前正在运行的容器**

```shell
#我们通常容器都是使用后台方式运行的，需要进入容器，修改一些配置

#命令
docker exec -it 容器id  /bin/bash

#方式二 
docker attach 容器id 
正在执行当前的代码。。。

#docker exec # 进入容器后开启一个新的终端，可以在里面操作（常用）
# docker attach # 进入容器正在执行的终端，不会启动新的进程！
```



从容器中拷贝文件到主机上

```shell
docker cp 容器id：容器内路径  主机路径
[root@hadoop01 home]# docker cp 34e483a845c4:/home/test.java /home


# 容器中
[root@34e483a845c4 /]# ls        
bin  etc   lib	  lost+found  mnt  proc  run   srv  tmp  var
dev  home  lib64  media       opt  root  sbin  sys  usr
[root@34e483a845c4 /]# cd home 
[root@34e483a845c4 home]# ls
[root@34e483a845c4 home]# touch test.java
[root@34e483a845c4 home]# ls
test.java
[root@34e483a845c4 home]# exit     
exit
[root@hadoop01 home]# docker ps -a
CONTAINER ID   IMAGE     COMMAND       CREATED         STATUS                     PORTS     NAMES
34e483a845c4   centos    "/bin/bash"   2 minutes ago   Exited (0) 3 seconds ago             interesting_saha
#将文件拷贝出来到主机上
[root@hadoop01 home]# docker cp 34e483a845c4:/home/test.java /home
[root@hadoop01 home]# ls
hello.java  master  test.java
[root@hadoop01 home]# 
```

## 小结

![]( https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220223192922.png)

```shell
attach    Attach to a running container  #当前shell下attach连接指定运行镜像
build     Build an image from a Dockerfile  #通过Dockerfile定制镜像
commit    Create a new image from a container's changes  #提交当前容器为新的镜像
cp    Copy files/folders from a container to a HOSTDIR or to STDOUT  #从容器中拷贝指定文件或者目录到宿主机中
create    Create a new container  #创建一个新的容器，同run 但不启动容器
diff    Inspect changes on a container's filesystem  #查看docker容器变化
events    Get real time events from the server#从docker服务获取容器实时事件
exec    Run a command in a running container#在已存在的容器上运行命令
export    Export a container's filesystem as a tar archive  #导出容器的内容流作为一个tar归档文件(对应import)
history    Show the history of an image  #展示一个镜像形成历史
FullHouse'p13 的命令 3
rename    Rename a container  #重命名容器
restart    Restart a running container  #重启运行的容器
rm    Remove one or more containers  #移除一个或者多个容器
rmi    Remove one or more images  #移除一个或多个镜像(无容器使用该镜像才可以删除，否则需要删除相关容器才可以继续或者-f强制删除)
run    Run a command in a new container  #创建一个新的容器并运行一个命令
save    Save an image(s) to a tar archive#保存一个镜像为一个tar包(对应load)
search    Search the Docker Hub for images  #在docker
hub中搜索镜像
start    Start one or more stopped containers#启动容器
stats    Display a live stream of container(s) resource usage statistics  #统计容器使用资源
stop    Stop a running container  #停止容器
tag         Tag an image into a repository  #给源中镜像打标签
top       Display the running processes of a container #查看容器中运行的进程信息
unpause    Unpause all processes within a container  #取消暂停容器
version    Show the Docker version information#查看容器版本号
wait         Block until a container stops, then print its exit code  #截取容器停止时的退出状态值
```



# **部署nginx**

```shell
# 1.搜索镜像 search 建议去docker搜索，可以看到帮助文档
# 2.下载镜像 docker pull nginx
# 3.测试nginx
[root@hadoop01 home]# docker images
REPOSITORY    TAG       IMAGE ID       CREATED        SIZE
mysql         latest    17b062d639f4   5 days ago     519MB
nginx         latest    c316d5a335a5   4 weeks ago    142MB
hello-world   latest    feb5d9fea6a5   5 months ago   13.3kB
centos        latest    5d0da3dc9764   5 months ago   231MB
# --name 容器名字
# -d 后台运行
# -p暴露端口 -p 宿主机端口：容器端口 
[root@hadoop01 home]# docker run -d --name nginx01 -p 3344:80 nginx
WARNING: IPv4 forwarding is disabled. Networking will not work.
a648a43efb72ffd299d32b986fc8d2969f1736f6b6027602cc98140ae231ce70
[root@hadoop01 home]# curl localhost:3344
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>
<p><em>Thank you for using nginx.</em></p>
</body>
</html>

[root@hadoop01 home]# docker exec -it nginx01 /bin/bash
root@a648a43efb72:/# ls
bin   dev		   docker-entrypoint.sh  home  lib64  mnt  proc  run   srv  tmp  var
boot  docker-entrypoint.d  etc			 lib   media  opt  root  sbin  sys  usr
root@a648a43efb72:/# whereis nginx
nginx: /usr/sbin/nginx /usr/lib/nginx /etc/nginx /usr/share/nginx
root@a648a43efb72:/# 


```

**bug：**

虚拟机上可以通过curl 访问docker上的nginx但是 不能通过浏览器访问

**解决办法：一·**

1、添加新的防火墙规则，开放对应的端口

2、iptables 问题，设置将数据转发到本机的其他网卡设备上就可以了。命令为：iptables -P FORWARD ACCEPT

3.重启docker，命令 ：systemctl start docker



**解决办法二：**

```shell
# vi /etc/sysctl.conf
添加代码：
net.ipv4.ip_forward=1
 
重启network服务
# systemctl restart network
 
查看是否修改成功
# sysctl net.ipv4.ip_forward
 
如果返回为“net.ipv4.ip_forward = 1”则表示成功了
```



# **部署tomcat**

```shell
#官方使用
docker run -d -p 3355:8080 --name tomcat01 tomcat
#我们之前的启动都是后台启动，停止容器之后，容器还是可以查到的， docker run -it --rm 一般用来测试，用完就删除


#测试访问没有问题 在外网浏览器是访问不到的

#进入容器 ，发现tomcat是少了东西的
docker exec -it 3caea0dfce8c /bin/bash

#发现问题 1：linxu命令少了。2没有webapps   阿里云镜像的原因，默认是最小的镜像，所有不必要的都兜除掉
#保证最小的
在容器中输入
cp -r webapps.dist/* webapps
就可以玩访问到了
```



# 可视化

* portainer（先用这个）

什么是portainer？

Docker图形化界面管理工具！ 提供一个后端面板供我们操作！

```shell
#安装图形化界面
docker run -d -p 8088:9000 --restart=always -v /var/run/docker.sock:/var/run/docker.sock --privileged=true portainer/portainer
```

![]( https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220223214935.png)

我设置的密码为:12345678

![]( https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220223215308.png)

# Docker镜像详解

## **什么是镜像**

镜像是一种轻量级、可执行的独立软件包，用来打包软件运行环境和基于运行环境开发的软件，它包含运行某个软件所需要的所有内容，包括代码，运行时（一个程序在运行或者在被执行的依赖）、库，环境变量和配置文件。

如何获得镜像：

* 从远程仓库下载
* 朋友拷贝给你
* 自己制作镜像 DockerFile

## Docker镜像加载原理

Docker的镜像实际上由一层一层的文件系统组成，这种层级的文件系统是UnionFS联合文件系统。

![]( https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220223215412.png)

![]( https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220223215428.png)

对于一个精简的OS，rootfs可以很小，只需要包括最基本的命令、工具和程序库就可以了，因为底层直接用Host的kernel，自己只需要提供rootfs就行了。由此可见对于不同的linux发行版，bootfs基本是一致的，rootfs会有差别，因此不同的发行版可以共用bootfs。

# 分层理解

![]( https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220223221645.png)

所有对容器的改动，无论添加、删除、还是修改文件都只会发生在容器层中。只有容器层是可写的，容器层下面的所有镜像层都是只读的。

![]( https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220223221937.png)

# Commit镜像

```shell
docker commit 提交容器成为一个新的副本
#命令和git原理类似
docker commit -m="提交的描述信息" -a="作者" 容器id 目标镜像名：[TAG]
```



## 实战测试：

```shell
# 启动一个默认的tomcat
[root@hadoop01 ~]# docker exec -it tomcat01 /bin/bash
root@d8ebf24788f9:/usr/local/tomcat# ls
BUILDING.txt	 LICENSE  README.md	 RUNNING.txt  conf  logs	    temp     webapps.dist
CONTRIBUTING.md  NOTICE   RELEASE-NOTES  bin	      lib   native-jni-lib  webapps  work

#发现这个默认的tomcat 是没有webapps应用 ，镜像的原因：官方的镜像默认webapps下面是没有文件的！

#我自己拷贝进去了基本文件
root@d8ebf24788f9:/usr/local/tomcat# cp -r webapps.dist/ webapps
# 将我们操作过的容器通过commit提交为一个镜像
docker commit -a="zhang" -m="add webapps app" d8ebf24788f9 tomcat02:1.0
# 查看我们自己的镜像
[root@hadoop01 ~]# docker images
REPOSITORY            TAG       IMAGE ID       CREATED              SIZE
tomcat02              1.0       4e4fd43e89dd   About a minute ago   684MB
mysql                 latest    17b062d639f4   5 days ago           519MB
tomcat                9.0       1fd0bf15d6bd   11 days ago          680MB
nginx                 latest    c316d5a335a5   4 weeks ago          142MB
portainer/portainer   latest    580c0e4e98b0   11 months ago        79.1MB

```

