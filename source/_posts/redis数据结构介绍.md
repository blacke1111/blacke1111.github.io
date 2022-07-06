---
title: redis数据结构介绍
date: 2022-05-11 20:17:07
categories: Redis
---

# Redis数据结构介绍

## Redis通用命令

* keys: 查看符合模板的所有key , 不建议在生产环境设备上使用

  ```shell
  #查看所有的key
  keys *
  ```

* 删除一个key

  ```shell
  #删除key为name的数据
  del name
  ```

* EXISTS: 查看一个key是否存在

  ```shell
  #查看name的key是否存在
  exists name
  ```

* EXPIRE: 给一个key设置有效期，有效期到期时该key会被自动删除

  ```shell
  #设置name的有效期为10秒 作用在已经已经存在的key上，而不是新建一个key
  EXPIRE name 10
  ```

* TTL：查看一个KEY的剩余有效期

  ```shell
  #查看name的有效期还剩多少
  TTL name
  ```

## String类型

String类型，也就是字符串类型，是Redis中最简单的存储类型。
其value是字符串，不过根据字符串的格式不同，又可以分为3类：

* string：普通字符串
* int：整数类型，可以做自增、自减操作
* float：浮点类型，可以做自增、自减操作

### String常见命令

String的常见命令有：

* SET：添加或者修改已经存在的一个String类型的键值对
* GET：根据key获取String类型的value
* MSET：批量添加多个String类型的键值对
* MGET：根据多个key获取多个String类型的value
* INCR：让一个整型的key自增1 **作用在已经存在的key上，让他的value值自增1**
* INCRBY:让一个整型的key自增并指定步长，例如：incrby num 2 让num值自增2
* INCRBYFLOAT：让一个浮点类型的数字自增并指定步长
* SETNX：添加一个String类型的键值对，前提是这个key不存在，否则不执行 与 set key value nx 效果一样
* SETEX：添加一个String类型的键值对，并且指定有效期 对应 通用命令中的expire

### 思考

Redis没有类似MySQL中的Table的概念，我们该如何区分不同类型的key呢？

* 例如，需要存储用户、商品信息到redis，有一个用户id是1，有一个商品id恰好也是1 

Redis的key允许有多个单词形成层级结构，多个单词之间用':'隔开，格式如下：

![image-20220511205015062](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220511205015062.png)

例如我们的项目名称叫 heima，有user和product两种不同类型的数据，我们可以这样定义key：

* user相关的key：project:user:1
* product相关的key：project:product:1 

![image-20220511211029460](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220511211029460.png)

可以看到在客户端中形成了层级关系

![image-20220511210945020](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220511210945020.png)

## Hash类型

Hash类型，也叫散列，其value是一个无序字典，类似于Java中的HashMap结构。
String结构是将对象序列化为JSON字符串后存储，当需要修改对象某个字段时很不方便：

![image-20220511211242749](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220511211242749.png)

Hash结构可以将对象中的每个字段独立存储，可以针对单个字段做CRUD：

![image-20220511211255996](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220511211255996.png)

### Hash的常见命令有：

Hash的常见命令有：

* HSET key field value：添加或者修改hash类型key的field的值
* HGET key field：获取一个hash类型key的field的值
* HMSET：批量添加多个hash类型key的field的值
* HMGET：批量获取多个hash类型key的field的值
* HGETALL：获取一个hash类型的key中的所有的field和value
* HKEYS：获取一个hash类型的key中的所有的field
* HVALS：获取一个hash类型的key中的所有的value
* HINCRBY:让一个hash类型key的字段值自增并指定步长
* HSETNX：添加一个hash类型的key的field值，前提是这个field不存在，否则不执行

## List类型

Redis中的List类型与Java中的LinkedList类似，可以看做是一个双向链表结构。既可以支持正向检索和也可以支持反向检索。
特征也与LinkedList类似：

* 有序
* 元素可以重复
* 插入和删除快
* 查询速度一般
  常用来存储一个有序数据，例如：朋友圈点赞列表，评论列表等。

### List的常见命令有：

* LPUSH key element ... ：向列表左侧插入一个或多个元素
* LPOP key：移除并返回列表左侧的第一个元素，没有则返回nil
* RPUSH key element ... ：向列表右侧插入一个或多个元素
* RPOP key：移除并返回列表右侧的第一个元素
* LRANGE key star end：返回一段角标范围内的所有元素  从0开始 左闭 右闭 范围（[start,end]）
* BLPOP和BRPOP：与LPOP和RPOP类似，只不过在没有元素时等待指定时间，而不是直接返回nil （**B代表阻塞**）

![image-20220511212050177](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220511212050177.png)

如何利用List结构模拟一个栈?
**• 入口和出口在同一边**
如何利用List结构模拟一个队列?
**• 入口和出口在不同边**
如何利用List结构模拟一个阻塞队列?
**• 入口和出口在不同边**
**• 出队时采用BLPOP或BRPOP**

## Set类型

Redis的Set结构与Java中的HashSet类似，可以看做是一个value为null的HashMap。因为也是一个hash表，因此具备与
HashSet类似的特征：

* 无序
* 元素不可重复
* 查找快
* 支持交集、并集、差集等功能

### Set的常见命令有：

* SADD key member ... ：向set中添加一个或多个元素
* SREM key member ... : 移除set中的指定元素
* SCARD key： 返回set中元素的个数
* SISMEMBER key member：判断一个元素是否存在于set中
* SMEMBERS：获取set中的所有元素
* SINTER key1 key2 ... ：求key1与key2的交集
* SDIFF key1 key2 ... ：求key1与key2的差集
* SUNION key1 key2 ..：求key1和key2的并集



Set命令的练习 （待完善）
将下列数据用Redis的Set集合来存储：

* 张三的好友有：李四、王五、赵六
* 李四的好友有：王五、麻子、二狗

利用Set的命令实现下列功能：

* 计算张三的好友有几人
* 计算张三和李四有哪些共同好友
* 查询哪些人是张三的好友却不是李四的好友
* 查询张三和李四的好友总共有哪些人
* 判断李四是否是张三的好友
* 判断张三是否是李四的好友
* 将李四从张三的好友列表中移除

## SortedSet类型

Redis的SortedSet是一个可排序的set集合，与Java中的TreeSet有些类似，但底层数据结构却差别很大。SortedSet中的每一个元素都带有一个score属性，可以基于score属性对元素排序，底层的实现是一个跳表（SkipList）加 hash表

SortedSet具备下列特性：

* 可排序
* 元素不重复
* 查询速度快

因为SortedSet的可排序特性，经常被用来实现排行榜这样的功能。

### SortedSet的常见命令有：

* ZADD key score member：添加一个或多个元素到sorted set ，如果已经存在则更新其score值
* ZREM key member：删除sorted set中的一个指定元素
* ZSCORE key member : 获取sorted set中的指定元素的score值
* ZRANK key member：获取sorted set 中的指定元素的排名
* ZCARD key：获取sorted set中的元素个数
* ZCOUNT key min max：统计**score值**在给定范围内的所有元素的个数 **min,max 对应的是score值**
* ZINCRBY key increment member：让sorted set中的指定元素自增，步长为指定的increment值
* ZRANGE key min max：按照score排序后，获取指定排名范围内的元素 **min,max 对应的是索引值**
* ZRANGEBYSCORE key min max：按照score排序后，获取指定score范围内的元素 **min,max 对应的是score值**
* ZDIFF、ZINTER、ZUNION：求差集、交集、并集

注意：所有的排名默认都是升序，**如果要降序则在命令的Z后面添加REV即可**

### 案例练习:

将班级的下列学生得分存入Redis的SortedSet中：
Jack 85, Lucy 89, Rose 82, Tom 95, Jerry 78, Amy 92, Miles 76

```shell
zadd stus 85 jack 89 lucy 82 rose 95 tom 78 jerry 92 amy 76 miles
```

• 并实现下列功能：
• 删除Tom同学

```shell
zrem stus tom
```

• 获取Amy同学的分数

```shell
zscore stus amy
```

• 获取Rose同学的排名

```shell
#从高到底排名 0开始
zrevrank stus rose
```

• 查询80分以下有几个学生

```shell
zcount stus 0 80
```

• 给Amy同学加2分

```shell
zincrby stus 2 amy
```

• 查出成绩前3名的同学

```shell
zrevrange stus 0 2
```

• 查出成绩80分以下的所有同学

```shell
zrangebyscore stus 0 80
```

## GEO数据结构

GEO就是Geolocation的简写形式，代表地理坐标。Redis在3.2版本中加入了对GEO的支持，允许存储地理坐标信息，帮助
我们根据经纬度来检索数据。常见的命令有：

* GEOADD：添加一个地理空间信息，包含：经度（longitude）、纬度（latitude）、值（member）
* GEODIST：计算指定的两个点之间的距离并返回
* GEOHASH：将指定member的坐标转为hash字符串形式并返回
* GEOPOS：返回指定member的坐标
* GEORADIUS：指定圆心、半径，找到该圆内包含的所有member，并按照与圆心之间的距离排序后返回。6.2以后已
  废弃
* GEOSEARCH：在指定范围内搜索member，并按照与指定点之间的距离排序后返回。范围可以是圆形或矩形。6.2.新
  功能
* GEOSEARCHSTORE：与GEOSEARCH功能一致，不过可以把结果存储到一个指定的key。 6.2.新功能



## BitMap 数据结构

Redis中是利用string类型数据结构实现BitMap，因此最大上限是512M，转换为bit则是 2^32个bit位。
BitMap的操作命令有：

* SETBIT：向指定位置（offset）存入一个0或1
* GETBIT ：获取指定位置（offset）的bit值
* BITCOUNT ：统计BitMap中值为1的bit位的数量
* BITFIELD ：操作（查询、修改、自增）BitMap中bit数组中的指定位置（offset）的值
* BITFIELD_RO ：获取BitMap中bit数组，并以十进制形式返回
* BITOP ：将多个BitMap的结果做位运算（与 、或、异或）
* BITPOS ：查找bit数组中指定范围内第一个0或1出现的位置

![image-20220611164142503](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220611164142503.png)

第一位 从 左边算起 ，所以是1

```shell
# u3 表示 无符号 查询3位bit 从第0位开始（按从左到右的顺序）
BITFIELD  get bm1 u3 0
结果 ：6 二进制位:110
```



## HyperLogLog用法

首先我们搞懂两个概念：

* UV：全称Unique Visitor，也叫**独立访客量**，是指通过互联网访问、浏览这个网页的自然人。1天内同一个用户多次
  访问该网站，只记录1次。
* PV：全称Page View，也叫**页面访问量或点击量**，用户每访问网站的一个页面，记录1次PV，用户多次打开页面，则记
  录多次PV。往往用来衡量网站的流量。

UV统计在服务端做会比较麻烦，因为要判断该用户是否已经统计过了，需要将统计过的用户信息保存。但是如果每个访问的用户都保存到Redis中，数据量会非常恐怖。

![image-20220613123915888](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220613123915888.png)
