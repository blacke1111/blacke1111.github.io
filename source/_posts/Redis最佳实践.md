---
title: Redis最佳实践
date: 2023-02-26 16:55:29
categories: Redis
---

# 优雅的key结构

Redis的Key虽然可以自定义，但最好遵循下面的几个最佳实践约定：
遵循基本格式：[业务名称]:[数据名]:[id]
长度不超过44字节
不包含特殊字符
例如：我们的登录业务，保存用户信息，其key是这样的：
优点：

* 可读性强
* 避免key冲突
* 方便管理
* 更节省内存： key是string类型，底层编码包含int、embstr和raw三种。embstr在小于44字节使用，采用连续内存空间，内存占用更小

# 什么是BigKey

BigKey通常以Key的大小和Key中成员的数量来综合判定，例如：

* Key本身的数据量过大：一个String类型的Key，它的值为5 MB。

* Key中的成员数过多：一个ZSET类型的Key，它的成员数量为10,000个。

* Key中成员的数据量过大：一个Hash类型的Key，它的成员数量虽然只有1,000个但这些成员的Value（值）总大小为100 MB。


  推荐值：
  单个key的value小于10KB
  对于集合类型的key，建议元素数量小于1000

## BigKey的危害

* 网络阻塞

对BigKey执行读请求时，少量的QPS就可能导致带宽使用率被占满，导致Redis实例，乃至所在物理机变慢

*  数据倾斜

BigKey所在的Redis实例内存使用率远超其他实例，无法使数据分片的内存资源达到均衡

*  Redis阻塞

对元素较多的hash、list、zset等做运算会耗时较旧，使主线程被阻塞

* CPU压力

对BigKey的数据序列化和反序列化会导致CPU的使用率飙升，影响Redis实例和本机其它应用



## 如何发现Bigkey

* redis-cli --bigkeys

利用redis-cli提供的--bigkeys参数，可以遍历分析所有key，并返回Key的整体统计信息与每个数据的Top1的big key

* scan扫描

自己编程，利用scan扫描Redis中的所有key，利用strlen、hlen等命令判断key的长度（此处不建议使用MEMORY USAGE）

* 第三方工具

利用第三方工具，如 Redis-Rdb-Tools 分析RDB快照文件，全面分析内存使用情况
 网络监控

* 自定义工具，监控进出Redis的网络数据，超出预警值时主动告警

## 如何删除BigKey

BigKey内存占用较多，即便时删除这样的key也需要耗费很长时间，导致Redis主线程阻塞，引发一系列问题。

* redis 3.0 及以下版本
  如果是集合类型，则遍历BigKey的元素，先逐个删除子元素，最后删除BigKey
* Redis 4.0以后
   Redis在4.0后提供了异步删除的命令：unlink



例1：比如存储一个User对象，我们有三种存储方式：
方式一：json字符串

![image-20220626184141387](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220626184141387.png)

方式二：字段打散

![image-20220626184154994](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220626184154994.png)

方式三：hash

![image-20220626184208714](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220626184208714.png)

例2：假如有hash类型的key，其中有100万对field和value，field是自增id，这个key存在什么问题？如何优化？

![image-20220626184327470](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220626184327470.png)

存在的问题：
hash的entry数量超过500时，会使用哈希表而不是ZipList，内存占用较多。

可以通过hash-max-ziplist-entries配置entry上限。但是如果entry过多就会导致BigKey问题

例2：假如有hash类型的key，其中有100万对field和value，field是自增id，这个key存在什么问题？如何优化？

方案二：拆分为string类型

![image-20220626184408180](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220626184408180.png)

存在的问题：
string结构底层没有太多内存优化，内存占用较多。
想要批量获取这些数据比较麻烦

例2：假如有hash类型的key，其中有100万对field和value，field是自增id，这个key存在什么问题？如何优化？

方案三：拆分为小的hash，将 id / 100 作为key， 将id % 100 作为field，这样每100个元素为一个Hash

![image-20220626184517656](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220626184517656.png)

```java
void testSmallHash(){
    int hashSize=100;
    Map(String,String) map=new HashMap(hashSize);
    for(int i=1;i<=100000;i++){
        int k=(i-1)/hashSize;
        int v=i%hashSize;
        map.put("key_"+v,"value_"+v);
        if(v== 0){
            jedis.hmset("test:small:hash_"+k,map);
        }
    }
}
```



Key的最佳实践：

* 固定格式：[业务名]:[数据名]:[id]
* 足够简短：不超过44字节
* 不包含特殊字符

Value的最佳实践：

* 合理的拆分数据，拒绝BigKey
* 选择合适数据结构
* Hash结构的entry数量不要超过1000
* 设置合理的超时时间

# 大量数据导入的方式

MSET

Redis提供了很多Mxxx这样的命令，可以实现批量插入数据，例如：

* mset
* hmset

注意:

不要在一次批处理中传输太多命令，否则单次命令占用带宽过多，会导致网络阻塞

利用mset批量插入10万条数据：

```java
@Test
void testMxx() {
    String[] arr = new String[2000];
    int j;
    for (int i = 1; i <= 100000; i++) {
        j = (i % 1000) << 1;
        arr[j] = "test:key_" + i;
        arr[j + 1] = "value_" + i;
        if (j == 0) {
            jedis.mset(arr);
        }
    }
}
```

## Pipeline

MSET虽然可以批处理，但是却只能操作部分数据类型，**因此如果有对复杂数据类型的批处理需要，建议使用Pipeline功能：**

```java
@Test
void testPipeline() {    // 创建管道    
    Pipeline pipeline = jedis.pipelined();
    for (int i = 1; i <= 100000; i++) {        // 放入命令到管道        
        pipeline.set("test:key_" + i, "value_" + i);
        if (i % 1000 == 0) {
            // 每放入1000条命令，批量执行            
            pipeline.sync();
        }
    }
}
```

批量处理的方案：

1. 原生的M操作
2. Pipeline批处理

注意事项：

1. 批处理时不建议一次携带太多命令
2. Pipeline的多个命令之间不具备原子性

## 集群下的批处理

如MSET或Pipeline这样的批处理需要在一次请求中携带多条命令，而此时如果Redis是一个集群，那批处理命令的多个key必须落在一个插槽中，否则就会导致执行失败。

![image-20220626192044176](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220626192044176.png)

Springboot 给我们做了封装 我们只需要使用 **redisTemplate.opsForlate().multiSet(map);** 底层使用了Nio做异步和多线程

# 持久化配置



Redis的持久化虽然可以保证数据安全，但也会带来很多额外的开销，因此持久化请遵循下列建议：

1. 用来做缓存的Redis实例尽量不要开启持久化功能
2. 建议关闭RDB持久化功能，使用AOF持久化
3. 利用脚本定期在slave节点做RDB，实现数据备份
4. 设置合理的rewrite阈值，避免频繁的bgrewrite
5. 配置no-appendfsync-on-rewrite = yes，禁止在rewrite期间做aof，避免因AOF引起的阻塞

部署有关建议：

1. Redis实例的物理机要预留足够内存，应对fork和rewrite
2. 单个Redis实例内存上限不要太大，例如4G或8G。可以加快fork的速度、减少主从同步、数据迁移压力
3. 不要与CPU密集型应用部署在一起
4. 不要与高硬盘负载应用一起部署。例如：数据库、消息队列





![image-20220626202449016](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220626202449016.png)



慢查询：在Redis执行时耗时超过某个阈值的命令，称为慢查询。

慢查询的阈值可以通过配置指定：

* slowlog-log-slower-than：慢查询阈值，单位是微秒。默认是10000，建议1000

慢查询会被放入慢查询日志中，日志的长度有上限，可以通过配置指定：

* slowlog-max-len：慢查询日志（本质是一个队列）的长度。默认是128，建议1000

![image-20220627125940496](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220627125940496.png)

![image-20220627125945247](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220627125945247.png)

查看慢查询日志列表：

* slowlog len：查询慢查询日志长度
* slowlog get [n]：读取n条慢查询日志
* slowlog reset：清空慢查询列表

# 命令及安全配置

Redis会绑定在0.0.0.0:6379，这样将会将Redis服务暴露到公网上，而Redis如果没有做身份认证，会出现严重的安全漏洞.
漏洞重现方式：https://cloud.tencent.com/developer/article/1039000

漏洞出现的核心的原因有以下几点：

* Redis未设置密码
* 利用了Redis的config set命令动态修改Redis配置
* 使用了Root账号权限启动Redis

为了避免这样的漏洞，这里给出一些建议：

* Redis一定要设置密码
* 禁止线上使用下面命令：keys、flushall、flushdb、config set等命令。可以利用rename-command禁用。
* bind：限制网卡，禁止外网网卡访问
* 开启防火墙
* 不要使用Root账户启动Redis
* 尽量不是有默认的端口

# 内存配置

当Redis内存不足时，可能导致Key频繁被删除、响应时间变长、QPS不稳定等问题。当内存使用率达到90%以上时就需要我们警惕，并快速定位到内存占用的原因。

![image-20220627132153747](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220627132153747.png)

Redis提供了一些命令，可以查看到Redis目前的内存分配状态：
* info memory
* memory xxx

内存缓冲区常见的有三种：
复制缓冲区：主从复制的repl_backlog_buf，如果太小可能导致频繁的全量复制，影响性能。通过repl-backlog-size来设置，默认1mb
AOF缓冲区：AOF刷盘之前的缓存区域，AOF执行rewrite的缓冲区。无法设置容量上限
客户端缓冲区：分为输入缓冲区和输出缓冲区，输入缓冲区最大1G且不能设置。输出缓冲区可以设置

![image-20220627133059831](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220627133059831.png)

![image-20220627133236120](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220627133236120.png)

查看所有客户端连接:
**client list**

# 集群完整性问题

集群虽然具备高可用特性，能实现自动故障恢复，但是如果使用不当，也会存在一些问题：

1. 集群完整性问题
2. 集群带宽问题
3. 数据倾斜问题
4. 客户端性能问题
5. 命令的集群兼容性问题
6. lua和事务问题

在Redis的默认配置中，如果发现任意一个插槽不可用，则整个集群都会停止对外服务：

![image-20220627134200185](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220627134200185.png)

为了保证高可用特性，这里建议将 cluster-require-full-coverage配置为false



集群节点之间会不断的互相Ping来确定集群中其它节点的状态。每次Ping携带的信息至少包括：

* 插槽信息
* 集群状态信息

集群中节点越多，集群状态信息数据量也越大，10个节点的相关信息可能达到1kb，此时每次集群互通需要的带宽会非常高。
解决途径：

1. 避免大集群，集群节点数不要太多，最好少于1000，如果业务庞大，则建立多个集群。
2. 避免在单个物理机中运行太多Redis实例
3. 配置合适的cluster-node-timeout值
