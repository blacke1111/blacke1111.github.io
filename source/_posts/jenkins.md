---
title: jenkins
date: 2022-03-30 15:35:17
categories: jenkins
---





## 手动打包

### 创建一个普通的spring boot工程

我用的是之前docker时测试的一个工程，就是一个最简单的web工程 就一个接口

![image-20220330154746974](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220330154746974.png)

![image-20220330154657657](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220330154657657.png)

### 工程进行打包：

打开idea的命令行：

![image-20220330154941176](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220330154941176.png)

使用maven进行打包

```shell
#执行
mvn clean package
```

打包完成后：

进入项目中target陌路，找到打包完成后的jar包，就好了

# 



配置java环境（略）

配置maven环境：

上传maven压缩包到linux上。

![image-20220330162640990](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220330162640990.png)

**第二步：解压安装包**

tar -zxvf 你自己解压包名称
**第三步：建立软连接**
ln -s /usr/local/apache-maven-3.6.1/  /usr/local/maven

**对应你自己的地址**

第四步：修改环境变量
vim /etc/profile
export MAVEN_HOME=/usr/local/maven
export PATH=$PATH:$MAVEN_HOME/bin

通过命令source /etc/profile让profile文件立即生效
source /etc/profile

第五步、测试是否安装成功
mvn –v

安装git环境：（略）

# jenkins安装：

## 第一步：上传或下载安装包

cd/usr/local/software/jenkins
jenkins.war

第二步：启动
nohup java -jar /usr/local/software/jenkins.war >/usr/local/software/jenkins.out &

![image-20220330165059983](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220330165059983.png)

出现这个再次输入回车就可以了 ，启动可能更需要一点时间

![image-20220330165418591](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220330165418591.png)

进入这个就可以了

查看密码

![image-20220330165648194](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220330165648194.png)

![image-20220330165802142](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20220330165802142.png)

注意：配置国内的镜像
官方下载插件慢 更新下载地址

你的Jenkins工作目录在：/root/.jenkins

cd {你的Jenkins工作目录}/updates #进入更新配置位置

在目录下执行如下命令：

```
sed -i 's/http:\/\/updates.jenkins-ci.org\/download/https:\/\/mirrors.tuna.tsinghua.edu.cn\/jenkins/g' default.json && sed -i 's/http:\/\/www.google.com/https:\/\/www.baidu.com/g' default.json
```

这是直接修改的配置文件，如果前边Jenkins用sudo启动的话，那么这里的两个sed前均需要加上sudo重启Jenkins，安装插件

然后重启。

![image-20220330171941165](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220330171941165.png)

选择推荐安装

* 配置管理员密码

![image-20220330173020121](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220330173020121.png)



配置java目录：

![image-20220330174018361](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220330174018361.png)

查看自己的java 安装目录：

![image-20220330173928994](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220330173928994.png)

配置jdk：

![image-20220330200123709](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220330200123709.png)

配置maven：

![image-20220330174332247](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220330174332247.png)

配置git：

![image-20220330174737426](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220330174737426.png)

新建任务：

![image-20220330174827528](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20220330174827528.png)



gitee上新建仓库并上传项目到gitee：

![image-20220330175341411](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220330175341411.png)

配置git地址：

我配置了一个用户id为1

![image-20220330175601507](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220330175601507.png)



配置执行命令：

```shell
#!/bin/bash
#maven打包
mvn clean package
echo 'package ok!'
echo 'build start!'
cd ./
service_name="demodocker" #打包的名称
service_prot=8088
#查看镜像id
IID=$(docker images | grep "$service_name" | awk '{print $3}')
echo "IID $IID"
if [ -n "$IID" ]
then
    echo "exist $SERVER_NAME image,IID=$IID"
    #删除镜像
    docker rmi -f $service_name
    echo "delete $SERVER_NAME image"
    #构建
    docker build -t $service_name .
    echo "build $SERVER_NAME image"
else
    echo "no exist $SERVER_NAME image,build docker"
    #构建
    docker build -t $service_name .
    echo "build $SERVER_NAME image"
fi
#查看容器id
CID=$(docker ps | grep "$SERVER_NAME" | awk '{print $1}')
echo "CID $CID"
if [ -n "$CID" ]
then
    echo "exist $SERVER_NAME container,CID=$CID"
    #停止
    docker stop $service_name
    #删除容器
    docker rm $service_name
else
    echo "no exist $SERVER_NAME container"
fi
#启动
docker run -d --name $service_name --net=host -p $service_prot:$service_prot $service_name
#查看启动日志
docker logs -f  $service_name
```



tag: 

 jenkins运行任务对应的目录是：

![image-20220330204031442](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220330204031442.png)

test2022330 对应的是我gitee上的仓库名 

![image-20220330204105702](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220330204105702.png)

进入这个目录和git对比 完全一致

![image-20220330204131475](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220330204131475.png)

所以得出我们的Dockerfile 

 COPY为什么需要写成./target/*.jar

文件为：

```dockerfile
FROM java:8
COPY ./target/*.jar /app.jar
CMD ["--server.port=8088"]
EXPOSE 8088
ENTRYPOINT ["java","-jar","app.jar"]
```





控制台查看 启动微服务成功：

![image-20220330203920816](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220330203920816.png)

浏览器访问:

![image-20220330204531875](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220330204531875.png)



这只是打包一个测试项目。 多模块类似打包就可以了

下面是我自己遇到打包多模块是遇到的错误 ，找了好久中找到了一篇博客完美解决我的困扰。

# maven模块化项目总共模块相互引用打包失败问题解决

https://blog.csdn.net/liqi_q/article/details/80557157
