---

title: mysql的数据目录
date: 2022-04-04 20:34:15
categories: mysql
---

# MySQL8的主要目录结构



这是我自己指定的mysql 数据的存储的地方

![image-20220404203836567](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220404203836567.png)



可以看到我指定的存放位置对应的就上面的位置

![image-20220404204358589](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220404204358589.png)

数据库运行起来对应的文件：

![image-20220404204010084](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220404204010084.png)





## 配置文件目录

**linux下：**

配置文件目录：/usr/share/mysql-8.0（命令及配置文件），/etc/mysql（如my.cnf）

![image-20220404204833928](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220404204833928.png)

my.cnf就相当于windows 下的my。ini

![image-20220404204929562](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220404204929562.png)

# 数据库和文件系统的关系

## 查看默认数据库

查看一下在我的计算机上当前有哪些数据库：

```mysql
mysql> SHOW DATABASES;
```

* mysql

MySQL 系统自带的核心数据库，它存储了MySQL的用户账户和权限信息，一些存储过程、事件的定义信息，一些运行过程中产生的日志信息，一些帮助信息以及时区信息等。

* information_schema

MySQL 系统自带的数据库，这个数据库保存着MySQL服务器维护的所有其他数据库的信息，比如有哪些表、哪些视图、哪些触发器、哪些列、哪些索引。这些信息并不是真实的用户数据，而是一些描述性信息，有时候也称之为元数据。在系统数据库information_schema 中提供了一些以innodb_sys 开头的表，用于表示内部系统表。

* performance_schema

MySQL 系统自带的数据库，这个数据库里主要保存MySQL服务器运行过程中的一些状态信息，可以
用来监控 MySQL 服务的各类性能指标。包括统计最近执行了哪些语句，在执行过程的每个阶段都
花费了多长时间，内存的使用情况等信息。

* sys

MySQL 系统自带的数据库，这个数据库主要是通过视图的形式把information_schema 和
performance_schema 结合起来，帮助系统管理员和开发人员监控 MySQL 的技术性能。

## 数据库在文件系统中的表示

### InnoDB存储引擎模式

#### **mysql5.7下：**

进入存储目录:

cd /var/lib/mysql

再进入对应的数据库下:

![image-20220404205651334](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220404205651334.png)

**db.opt** 存储的我们数据库对应的比较规则和字符集等

**epm1.frm** 存储的是表结构 数据可以存放在**emp1.ibd**（独立表空间）下 页可以存放在系统表表空间下，他是独立于每个数据库的，emp1.ibd也可以称为独立表空间，

ibdate1 别名系统表空间 ，大小12MB，这个文件就是对应的系统表空间在文件系统上的表示。怎么才12M？注意这个文件是自扩展文件，当不够用的时候它会自己增加文件大小。

![image-20220404210128560](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220404210128560.png)

**独立表空间(file-per-table tablespace)**

在MySQL5.6.6以及之后的版本中，InnoDB并不会默认的把各个表的数据存储到系统表空间中，而是为每一个表建立一个独立表空间，也就是说我们创建了多少个表，就有多少个独立表空间。使用独立表空间来存储表数据的话，会在该表所属数据库对应的子目录下创建一个表示该独立表空间的文件，文件名和表名相同，只不过添加了一个.ibd 的扩展名而已，所以完整的文件名称长这样

**系统表空间与独立表空间的设置**

我们可以自己指定使用系统表空间还是独立表空间来存储数据，这个功能由启动参数innodb_file_per_table 控制，比如说我们想刻意将表数据都存储到系统表空间时，可以在启动MySQL服务器的时候这样配置：

```
[server]
innodb_file_per_table=0 # 0：代表使用系统表空间； 1：代表使用独立表空间
```

默认情况： 表示使用独立表空间

![image-20220404210646933](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220404210646933.png)

#### **mysql8.0下:**

如下看到他也有自己的系统表空间

![image-20220404210849579](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220404210849579.png)



进入数据库中，看到**8.0**没有.opt后缀名的文件，这不能说他没有保存每个数据库的字符集和比较规则，他把这些都存放到每个表的结构中去了。

![image-20220404212212984](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220404212212984.png)

**疑问：**

.frm的文件也没有了。 这是他把.frm和.ibd文件都存放到.ibd文件中了

这就需要解析ibd文件。oracle官方将frm文件的信息及更多信息移动到叫做序列化字典信息，SDI，SDI被写在ibd文件内部，Mysql8.0属于oracle旗下，oracle提供了一个应用程序ibd2sdi。

![image-20220404211857538](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220404211857538.png)

### MyISAM存储引擎模式

#### **mysql5.7下:**

先创建MyISAM存储引擎的数据表:

```mysql
CREATE TABLE `student_myisam` (
`id` bigint NOT NULL AUTO_INCREMENT,
`name` varchar(64) DEFAULT NULL,
`age` int DEFAULT NULL,
`sex` varchar(2) DEFAULT NULL,
PRIMARY KEY (`id`)
)ENGINE=MYISAM AUTO_INCREMENT=0 DEFAULT CHARSET=utf8mb3;
```



看到student_myisam表的文件有三个。

![image-20220404212707712](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220404212707712.png)

表结构：

在存储表结构方面， MyISAM 和InnoDB 一样，也是在数据目录下对应的数据库子目录下创建了一个专门用于描述表结构的文件：

```
表名.frm
```

表中数据和索引:

在MyISAM中的索引全部都是二级索引，该存储引擎的数据和索引是分开存放的。所以在文件系统中也是使用不同的文件来存储数据文件和索引文件，同时表数据都存放在对应的数据库子目录下。假如test表使用MyISAM存储引擎的话，那么在它所在数据库对应的atguigu 目录下会为test 表创建这三个文件：

```mysql
test.frm 存储表结构
#MYD和MYI 对应着InnoDB的ibd文件 可以看到索引和数据分开存储
test.MYD和 存储数据 (MYData)
test.MYI 存储索引 (MYIndex)
```

#### mysql8.0下：

sdi相当于5.7下的frm文件

![image-20220404213409091](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220404213409091.png)

# 小结：

![image-20220404213557250](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220404213557250.png)
