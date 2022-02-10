---
title: nginx高可用
date: 2022-01-13 20:06:37
categories: nginx
---





![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220113204020.png)

## 1,前提

（1）需要两台 nginx 服务器 （我用的是虚拟机centos7）
（2）需要 keepalived
（3）需要虚拟 ip

## 2、配置高可用的准备工作

（1）需要两台服务器 192.168.32.3 和 192.168.32.4
（2）在两台服务器安装 nginx
（3）在两台服务器安装 keepalived

## 3、在两台服务器安装 keepalived

（1）使用 yum 命令进行安装

```
yum install keepalived –y
```



查看keepalived是否安装成功  

```java
rpm -q -a keepalived
```

## 4、完成高可用配置（主从配置）

（1）修改/etc/keepalived/keepalived.conf 配置文件

主机配置：

```
! Configuration File for keepalived
global_defs {

    notification_email {

        acassen@firewall.loc

        failover@firewall.loc

        sysadmin@firewall.loc
     }

        notification_email_from Alexandre.Cassen@firewall.loc

        smtp_server 192.168.32.3

        smtp_connect_timeout 30

        router_id hadoop01 # 主机名字
}


vrrp_script chk_http_port {
        script "/usr/local/src/nginx_check.sh"
        interval 2 #（检测脚本执行的间隔）
        weight 2 # 权重
   }
vrrp_instance VI_1 {
    state MASTER # 备份服务器上将 MASTER 改为 BACKUP
    interface ens33 # 网卡
    virtual_router_id 51  # 主、备机的 virtual_router_id 必须相同
    priority 100 # 主、备机取不同的优先级，主机值较大，备份机值较小
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }

    virtual_ipaddress {
        192.168.32.50 # VRRP H 虚拟地址  这里一定还和你的虚拟机一个网段不然你的windows上访问不到。
    }

}
```

从机配置：

```
! Configuration File for keepalived
global_defs {

    notification_email {

        acassen@firewall.loc

        failover@firewall.loc

        sysadmin@firewall.loc
     }

        notification_email_from Alexandre.Cassen@firewall.loc

        smtp_server 192.168.32.3

        smtp_connect_timeout 30

        router_id hadoop01 # 主机名字
}


vrrp_script chk_http_port {
        script "/usr/local/src/nginx_check.sh"
        interval 2 #（检测脚本执行的间隔）
        weight 2 # 权重
   }
vrrp_instance VI_1 {
    state BACKUP # 备份服务器上将 MASTER 改为 BACKUP
    interface ens33 # 网卡
    virtual_router_id 51  # 主、备机的 virtual_router_id 必须相同
    priority 90 # 主、备机取不同的优先级，主机值较大，备份机值较小
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }

    virtual_ipaddress {
        192.168.32.50 # VRRP H 虚拟地址  这里一定还和你的虚拟机一个网段不然你的windows上访问不到。
    }

}

```



（2）在/usr/local/src 添加检测脚本

```

#!/bin/bash
A=`ps -C nginx –no-header |wc -l`
if [ $A -eq 0 ];then
 /usr/local/nginx/sbin/nginx
 sleep 2
 if [ `ps -C nginx --no-header |wc -l` -eq 0 ];then
 killall keepalived
 fi
fi
```





**把两台服务器上nginx和keepalived启动**
**启动nginx:** ./nginx
**启动keepalived**：systemctl start keepalived.service

## 5  输入 ip a查看虚拟地址映射

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220113221746.png)

## 6 windows上访问测试：

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20220113221820.png)
