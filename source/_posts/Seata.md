---
title: Seata
date: 2022-01-29 21:12:29
categories: springcloud
---





一次业务操作需要跨多个数据源或需要多个系统进行远程调用，就会产生分步式事务





## 安装:

解压官网下载的seate解压包。

进入conf目录：

先拷贝一份出厂默认的file.conf：

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220129213623.png)

主要修改：自定义事务组名称+事务日志存储模式为db+数据库连接信息



service模块修改事务组名称：

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220130201514.png)

存储模式为db

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220129214048.png)

修改数据库连接信息：

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220129220025.png)



建一个数据库：seata

导入conf下的db_store.sql文件建表

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220129214556.png)

**修改registry.conf文件：**

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220129215931.png)



我用的mysql8.0+所以seata需要在他的bin目录下添加你对应mysql版本的驱动包

**一定要把原来的删除掉**

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220129220056.png)

然后cmd启动bin下的seata-server.bat

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220129220159.png)



## 案例准备工作：

建库：

**seata_order:**

建表：

```sql
CREATE TABLE t_order(
    id BIGINT(11) NOT NULL AUTO_INCREMENT PRIMARY KEY ,
    user_id BIGINT(11) DEFAULT NULL COMMENT '用户id',
    product_id BIGINT(11) DEFAULT NULL COMMENT '产品id',
    count INT(11) DEFAULT NULL COMMENT '数量',
    money DECIMAL(11,0) DEFAULT NULL COMMENT '金额',
    status INT(1) DEFAULT NULL COMMENT '订单状态：0创建中，1已完结'
)ENGINE=InnoDB AUTO_INCREMENT=7 CHARSET=utf8;
```

**seata_storage**

建表：

```sql
CREATE TABLE t_storage(
    id BIGINT(11) NOT NULL AUTO_INCREMENT PRIMARY KEY ,
    product_id BIGINT(11) DEFAULT NULL COMMENT '产品id',
    total INT(11) DEFAULT NULL COMMENT '总库存',
    used INT(11) DEFAULT NULL COMMENT '已用库存',
    residue INT(11) DEFAULT NULL COMMENT '剩余库存'
)ENGINE=InnoDB AUTO_INCREMENT=7 CHARSET=utf8;
```

**seata_account**

建表：

```sql
CREATE TABLE t_account(
    id BIGINT(11) NOT NULL AUTO_INCREMENT PRIMARY KEY ,
    user_id BIGINT(11) DEFAULT NULL COMMENT '用户id',
    total DECIMAL(10,0) DEFAULT NULL COMMENT '总额度',
    used DECIMAL(10,0) DEFAULT NULL COMMENT '已用额度',
    residue DECIMAL(10,0) DEFAULT 0 COMMENT '剩余可用额度'
)ENGINE=InnoDB AUTO_INCREMENT=7 CHARSET=utf8;
```

创建回滚日志表：

在每个库中导入db_undo_log.sql，对应建表语句如下：

```sql
CREATE TABLE `undo_log` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `branch_id` bigint(20) NOT NULL,
  `xid` varchar(100) NOT NULL,
  `context` varchar(128) NOT NULL,
  `rollback_info` longblob NOT NULL,
  `log_status` int(11) NOT NULL,
  `log_created` datetime NOT NULL,
  `log_modified` datetime NOT NULL,
  `ext` varchar(100) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `ux_undo_log` (`xid`,`branch_id`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;
```

案例源码：https://gitee.com/haoyumaster/springcoud2022study



## AT模式：

第一阶段：

在一阶段，Seata 会拦截“业务 SQL”，
1  解析 SQL 语义，找到“业务 SQL”要更新的业务数据，在业务数据被更新前，将其保存成“before image”，
2  执行“业务 SQL”更新业务数据，在业务数据更新之后，
3  其保存成“after image”，最后生成行锁。
以上操作全部在一个数据库事务内完成，这样保证了一阶段操作的原子性。

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220130215035.png)



第二阶段：

二阶段如是顺利提交的话，
因为“业务 SQL”在一阶段已经提交至数据库，所以Seata框架只需将一阶段保存的快照数据和行锁删掉，完成数据清理即可。

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220130215204.png)

二阶段回滚：
二阶段如果是回滚的话，Seata 就需要回滚一阶段已经执行的“业务 SQL”，还原业务数据。
回滚方式便是用“before image”还原业务数据；但在还原前要首先要校验脏写，对比“数据库当前业务数据”和 “after image”，
如果两份数据完全一致就说明没有脏写，可以还原业务数据，如果不一致就说明有脏写，出现脏写就需要转人工处理。

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220130221150.png)

图上before image -》逆向sql  解释：

利用before image（前置快照） 生成逆向sql数据还原。。

如果高并发出现脏数据 就不能进行回滚 ，交由人工处理  ，因为已经有人在这个事务中动过数据库。
