---
title: docker高级
date: 2022-02-27 19:57:18
categories: Docker  
---

# Docker Compose

Compose 是 Docker 公司推出的一个工具软件，可以管理多个 Docker 容器组成一个应用。你需要定义一个 YAML 格式的配置文件docker-compose.yml，写好多个容器之间的调用关系。然后，只要一个命令，就能同时启动/关闭这些容器

## 能干嘛

docker建议我们每一个容器中只运行一个服务,因为docker容器本身占用资源极少,所以最好是将每个服务单独的分割开来但是这样我们又面临了一个问题？

如果我需要同时部署好多个服务,难道要每个服务单独写Dockerfile然后在构建镜像,构建容器,这样累都累死了,所以docker官方给我们提供了docker-compose多服务部署的工具

例如要实现一个Web微服务项目，除了Web服务容器本身，往往还需要再加上后端的数据库mysql服务容器，redis服务器，注册中心eureka，甚至还包括负载均衡容器等等。。。。。。

Compose允许用户通过一个单独的docker-compose.yml模板文件（YAML 格式）来定义一组相关联的应用容器为一个项目（project）。

可以很容易地用一个配置文件定义一个多容器的应用，然后使用一条指令安装这个应用的所有依赖，完成构建。Docker-Compose 解决了容器与容器之间如何管理编排的问题。

## 安装步骤：

官网地址：https://docs.docker.com/compose/install/

```shell
#下载 Docker Compose
curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
#对二进制文件应用可执行权限：
chmod +x /usr/local/bin/docker-compose
#测试是否安装成功
docker-compose --version

#卸载
rm /usr/local/bin/docker-compose

```

## Compose的核心概念

### 一文件

docker-compose.yml

### 二要素

服务：一个个应用容器实例，比如订单微服务，库存微服务，mysql容器等等

工程：由一组关联的应用容器组成的一个**完整业务单元**，在docker-compose.yml文件中定义

## Compose使用的三步骤

1.编写Dockerfile定义各个微服务应用并构建出对应的镜像文件

2.使用docker-compose.yml定义一个完整业务单元，安排好整体应用中的各个容器服务

3.最后,执行docker-compose up命令来启动并运行整个应用程序，完成一键部署上线

```shell
#常用命令
Compose常用命令
docker-compose -h                           # 查看帮助
docker-compose up                           # 启动所有docker-compose服务
docker-compose up -d                        # 启动所有docker-compose服务并后台运行
docker-compose down                         # 停止并删除容器、网络、卷、镜像。
docker-compose exec  yml里面的服务id                 # 进入容器实例内部  docker-compose exec docker-compose.yml文件中写的服务id /bin/bash
docker-compose ps                      # 展示当前docker-compose编排过的运行的所有容器
docker-compose top                     # 展示当前docker-compose编排过的容器进程

docker-compose logs  yml里面的服务id     # 查看容器输出日志
docker-compose config     # 检查配置
docker-compose config -q  # 检查配置，有问题才有输出
docker-compose restart   # 重启服务
docker-compose start     # 启动服务
docker-compose stop      # 停止服务

```





**案例编写spring-boot服务：**

我已上传到我的github：

https://github.com/blacke1111/mybatis-generator6

，打包docker_boot 服务生成对应jar包

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220228185519.png)



编写对应的Dockerfile文件：

```dockerfile
# 基础镜像使用java
FROM java:8
# 作者
MAINTAINER zzyy
# VOLUME 指定临时文件目录为/tmp，在主机/var/lib/docker目录下创建了一个临时文件并链接到容器的/tmp
VOLUME /tmp
# 将jar包添加到容器中并更名为zzyy_docker.jar
ADD docker_boot-0.0.1-SNAPSHOT.jar zzyy_docker.jar
# 运行jar包
RUN bash -c 'touch /zzyy_docker.jar'
ENTRYPOINT ["java","-jar","/zzyy_docker.jar"]
#暴露6001端口作为微服务
EXPOSE 6001
```

编写对应的compose文件：

```yaml
version: "3"

services:
  microService:
    image: zzyy_docker:1.6
    container_name: ms01
    ports:
      - "6001:6001"
    volumes:
      - /app/microService:/data
    networks:
      - atguigu_net
    depends_on:
      - redis
      - mysql

  redis:
    image: redis:6.0.8
    ports:
      - "6379:6379"
    volumes:
      - /app/redis/redis.conf:/etc/redis/redis.conf
      - /app/redis/data:/data
    networks:
      - atguigu_net
    command: redis-server /etc/redis/redis.conf

  mysql:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: '123456'
      MYSQL_ALLOW_EMPTY_PASSWORD: 'no'
      MYSQL_DATABASE: 'db2021'
      MYSQL_USER: 'zzyy'
      MYSQL_PASSWORD: 'zzyy123'
    ports:
      - "3306:3306"
    volumes:
      - /app/mysql/db:/var/lib/mysql
      - /app/mysql/conf/my.cnf:/etc/my.cnf
      - /app/mysql/init:/docker-entrypoint-initdb.d
    networks:
      - atguigu_net
    command: --default-authentication-plugin=mysql_native_password #解决外部无法访问

networks:
  atguigu_net:

```

上传到linux上：

在对应文件夹下执行：

```shell
docker-compose up -d
```

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220228190038.png)

可以看到服务已经启动完毕。、

进入mysql容器

```shell
docker exec -it mysql容器id bash
```

创建数据库db2021

```mysql
create database db2021;
```

在数据库db2021下建如下表：

```shell
CREATE TABLE `t_user` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `username` VARCHAR(50) NOT NULL DEFAULT '' COMMENT '用户名',
  `password` VARCHAR(50) NOT NULL DEFAULT '' COMMENT '密码',
  `sex` TINYINT(4) NOT NULL DEFAULT '0' COMMENT '性别 0=女 1=男 ',
  `deleted` TINYINT(4) UNSIGNED NOT NULL DEFAULT '0' COMMENT '删除标志，默认0不删除，1删除',
  `update_time` TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `create_time` TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  PRIMARY KEY (`id`)
) ENGINE=INNODB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8mb4 COMMENT='用户表';
```

测试：发送post请求

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220228190231.png)

然后查询对应用户：

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220228190301.png)



bug排查：

maven打包后找不到主属性清单

```
#修改pom配置为
 <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <executions>
                    <execution>
                        <goals>
                            <goal>repackage</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-resources-plugin</artifactId>
                <version>3.1.0</version>
            </plugin>
        </plugins>
    </build>
```





# Docker 容器监控

docker stats查看容器占用情况：

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220228191503.png)

通过docker stats命令可以很方便的看到当前宿主机上所有容器的CPU,内存以及网络流量等数据，一般小公司够用了。。。。

但是，docker stats统计结果只能是当前宿主机的全部容器，数据资料是实时的，没有地方存储、没有健康指标过线预警等功能

**CAdvisor监控收集+influxDB存储数据+Granfana展示图表**

## CAdvisor：

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220228191852.png)

## InfluxDB：

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220228191953.png)

## Granfana:

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220228192017.png)



## compose一键搭建平台



docker-compose.yml

```yml
version: '3.1'
 
volumes:
  grafana_data: {}
 
services:
 influxdb:
  image: tutum/influxdb:0.9
  restart: always
  environment:
    - PRE_CREATE_DB=cadvisor
  ports:
    - "8083:8083"
    - "8086:8086"
  volumes:
    - ./data/influxdb:/data
 
 cadvisor:
  image: google/cadvisor
  links:
    - influxdb:influxsrv
  command: -storage_driver=influxdb -storage_driver_db=cadvisor -storage_driver_host=influxsrv:8086
  restart: always
  ports:
    - "8080:8080"
  volumes:
    - /:/rootfs:ro
    - /var/run:/var/run:rw
    - /sys:/sys:ro
    - /var/lib/docker/:/var/lib/docker:ro
 
 grafana:
  user: "104"
  image: grafana/grafana
  user: "104"
  restart: always
  links:
    - influxdb:influxsrv
  ports:
    - "3000:3000"
  volumes:
    - grafana_data:/var/lib/grafana
  environment:
    - HTTP_USER=admin
    - HTTP_PASS=admin
    - INFLUXDB_HOST=influxsrv
    - INFLUXDB_PORT=8086
    - INFLUXDB_NAME=cadvisor
    - INFLUXDB_USER=root
    - INFLUXDB_PASS=root

```



执行：

```shell
docker-compose up -d
```

等待下载完毕。

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220228193024.png)

**浏览cAdvisor收集服务8080端口，influxdb存储服务8083端口，grafana展现服务300端口**



grafana配置数据源：

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220228193557.png)

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220228193816.png)

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220228194023.png)

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220228194006.png)

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220228194053.png)

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220228194158.png)

然后输入Title，sava保存

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220228194259.png)

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220228195848.png)
