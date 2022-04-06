---
title: mysql字符集
date: 2022-04-04 19:14:34
categories: mysql
---

# **字符集的相关操作**

**修改MySQL5.7字符集** 

## **修改步骤**

在MySQL 8.0版本之前，默认字符集为 latin1 ，utf8字符集指向的是 utf8mb3 。网站开发人员在数据库

设计的时候往往会将编码修改为utf8字符集。如果遗忘修改默认的编码，就会出现乱码的问题。从MySQL 

8.0开始，数据库的默认编码将改为 utf8mb4 ，从而避免上述乱码的问题。

### **操作1：查看默认使用的字符集**

```mysql
show variables like 'character%'; 
# 或者 
show variables like '%char%';
```



8.0.27下:

![image-20220404192215776](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220404192215776.png)

8.7.27下:

![image-20220404192235498](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220404192235498.png)



这就导致了我们在5.7.27 版本下添加 中文字符的时候会报错 ，需要修改它的字符集

### **操作2：修改字符集**

```shell
vim /etc/my.cnf
```

在MySQL5.7或之前的版本中，在文件最后加上中文字符集配置：

```
character_set_server=utf8
```

![image-20220404192528385](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220404192528385.png)

### **操作3**：重新启动**MySQL服务**

```shell
systemctl restart mysqld
```

tag:

> 但是原库、原表的设定不会发生变化，参数修改只对新建的数据库生效。

##  **已有库&表字符集的变更** 

```mysql
#修改已创建数据库的字符集
alter database dbtest1 character set 'utf8';
#修改已创建数据表的字符集
alter table t_emp convert to character set 'utf8';
```

> 注意：但是原有的数据如果是用非'utf8'编码的话，数据本身编码不会发生改变。已有数据需要导
>
> 出或删除，然后重新插入。

### **各级别的字符集**

MySQL有4个级别的字符集和比较规则，分别是

* 服务器级别

* 数据库级别

* 表级别

* 列级别

执行如下SQL语句：

```mysql
show variables like 'character%';
```

![image-20220404193406680](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220404193406680.png)

* character_set_server：服务器级别的字符集

* character_set_database：当前数据库的字符集

* character_set_client：服务器解码请求时使用的字符集

* character_set_connection：服务器处理请求时会把请求字符串从character_set_client转为

* character_set_connection 

* character_set_results：服务器向客户端返回数据时使用的字符集

 

#### **服务器级别** 

character_set_server ：服务器级别的字符集

我们可以在启动服务器程序时通过启动选项或者在服务器程序运行过程中使用 SET 语句修改这两个变量

的值。比如我们可以在配置文件中这样写：

```mysql
[server] character_set_server=gbk 
# 默认字符集 
collation_server=gbk_chinese_ci #对应的默认的比较规则
```

当服务器启动的时候读取这个配置文件后这两个系统变量的值便修改了。

 

#### **数据库级别** 

character_set_database ：当前数据库的字符集

我们在创建和修改数据库的时候可以指定该数据库的字符集和比较规则，具体语法如下：

```mysql
CREATE DATABASE 数据库名 
	[[DEFAULT] CHARACTER SET 字符集名称] 
	[[DEFAULT] COLLATE 比较规则名称]; 
ALTER DATABASE 数据库名 
	[[DEFAULT] CHARACTER SET 字符集名称] 
	[[DEFAULT] COLLATE 比较规则名称];
```

#### **表级别**

我们也可以在创建和修改表的时候指定表的字符集和比较规则，语法如下：

```mysql
CREATE TABLE 表名 (列的信息) 
	[[DEFAULT] CHARACTER SET 字符集名称] 
	[COLLATE 比较规则名称]] 

ALTER TABLE 表名 
	[[DEFAULT] CHARACTER SET 字符集名称] 
	[COLLATE 比较规则名称]
```

**如果创建和修改表的语句中没有指明字符集和比较规则，将使用该表所在数据库的字符集和比较规则作**

**为该表的字符集和比较规则。**

#### **列级别**

对于存储字符串的列，同一个表中的不同的列也可以有不同的字符集和比较规则。我们在创建和修改列

定义的时候可以指定该列的字符集和比较规则，语法如下：

```mysql
CREATE TABLE 表名( 
    列名 字符串类型 [CHARACTER SET 字符集名称] [COLLATE 比较规则名称], 其他列... 
);ALTER TABLE 表名 MODIFY 列名 字符串类型 [CHARACTER SET 字符集名称] [COLLATE 比较规则名称];
```

**对于某个列来说，如果在创建和修改的语句中没有指明字符集和比较规则，将使用该列所在表的字符集**

**和比较规则作为该列的字符集和比较规则。**

> 提示
>
> 在转换列的字符集时需要注意，如果转换前列中存储的数据不能用转换后的字符集进行表示会发生
>
> 错误。比方说原先列使用的字符集是utf8，列中存储了一些汉字，现在把列的字符集转换为ascii的
>
> 话就会出错，因为ascii字符集并不能表示汉字字符。

**字符集与比较规则**

**utf8** **与** **utf8mb4** 

utf8 字符集表示一个字符需要使用1～4个字节，但是我们常用的一些字符使用1～3个字节就可以表示

了。而字符集表示一个字符所用的最大字节长度，在某些方面会影响系统的存储和性能，所以设计

MySQL的设计者偷偷的定义了两个概念：

**utf8** ：字母abc...用一个字节表示, 一个中文字 用3字节表示

* **utf8mb3** ：阉割过的 utf8 字符集，只使用1～3个字节表示字符。

* **utf8mb4** ：正宗的 utf8 字符集，使用1～4个字节表示字符。

 在mysql中utf8是utf8mb3的别名,所以之后再mysql中提到utf8就意味着使用1~3个字节表示一个字符。

如果大家使用45字节编码一个字符的情况，比如存储一些emoji的表情，那请使用utf8mb4

```mysql
show charset;
#或者
show character set;
```

![image-20220404194832538](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220404194832538.png)

**比较规则**

上表中，MySQL版本一共支持41种字符集，其中的 Default collation 列表示这种字符集中一种默认的比较规则，里面包含着该比较规则主要作用于哪种语言，比如 utf8_polish_ci 表示以波兰语的规则比较， utf8_spanish_ci 是以西班牙语的规则比较， utf8_general_ci 是一种通用的比较规则。

后缀表示该比较规则是否区分语言中的重音、大小写。具体如下：

![image-20220404194955180](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220404194955180.png)

最后一列 Maxlen ，它代表该种字符集表示一个字符最多需要几个字节

**常用操作**

```mysql
#查看 GBK字符集的比较规则 
SHOW COLLATION LIKE 'gbk%'; 
#查看UTF-8字符集的比较规则 
SHOW COLLATION LIKE 'utf8%';
#查看服务器的字符集和比较规则
SHOW VARIABLES LIKE '%_server'; 
#查看数据库的字符集和比较规则 
SHOW VARIABLES LIKE '%_database'; 
#查看具体数据库的字符集 
SHOW CREATE DATABASE dbtest1; 
#修改具体数据库的字符集 
ALTER DATABASE dbtest1 DEFAULT CHARACTER SET 'utf8' COLLATE 'utf8_general_ci';

#查看表的字符集 
show create table employees; 
#查看表的比较规则 
show table status from atguigudb like 'employees'; 
#修改表的字符集和比较规则 
ALTER TABLE emp1 DEFAULT CHARACTER SET 'utf8' COLLATE 'utf8_general_ci';
```

> utf8_unicode_ci和utf8_general_ci对中，英文来说没有实质的差别。
>
> utf8_general_ci 校对速度快,但准确度较差
>
> utf8_unicode_ci 准确度高 但校对熟读稍慢
>
> 一般情况，用utf8_general_ci就够了。但如果有德语，法语或者俄语，请一定使用utf8_unicode_ci

**请求到响应过程中字符集的变化**

“gbk中文2个字节,英文1个字节;utf-8中文3个字节,英文1个字节”

![image-20220404200342955](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220404200342955.png)

1.现在假设我们客户端发送的请求是下边这个字符串：

```mysql
SELECT * FROM t WHERE s = '我';
```

为了方便大家理解这个过程，我们只分析字符 '我' 在这个过程中字符集的转换。

现在看一下在请求从发送到结果返回过程中字符集的变化：

客户端发送请求所使用的字符集

一般情况下客户端所使用的字符集和当前操作系统一致，不同操作系统使用的字符集可能不一样，如下：

* 类 Unix 系统使用的是 utf8  
* Windows 使用的是 gbk

2.当客户端使用的是 utf8 字符集，字符 '我' 在发送给服务器的请求中的字节形式就是：0xE68891服务器接收到客户端发送来的请求其实是一串二进制的字节，它会认为这串字节采用的字符集是character_set_client ，然后把这串字节转换为 character_set_connection 字符集编码的字符。

由于我的计算机上 character_set_client 的值是 utf8 ，首先会按照 utf8 字符集对字节串0xE68891 进行解码，得到的字符串就是 '我' ，然后按照 character_set_connection 代表的字符集，也就是 gbk 进行编码，得到的结果就是字节串 0xCED2 。 

3.因为表 t 的列 col 采用的是 gbk 字符集，与 character_set_connection 一致，所以直接到列中找字节值为 0xCED2 的记录，最后找到了一条记录。

4.上一步骤找到的记录中的 col 列其实是一个字节串 0xCED2 ， col 列是采用 gbk 进行编码的，所以首先会将这个字节串使用 gbk 进行解码，得到字符串 '我' ，然后再把这个字符串使用character_set_results 代表的字符集，也就是 utf8 进行编码，得到了新的字节串：0xE68891 ，然后发送给客户端。

5.由于客户端是用的字符集是 utf8 ，所以可以顺利的将 0xE68891 解释成字符 我 ，从而显示到我们的显示器上，所以我们人类也读懂了返回的结果。

![image-20220404201040917](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220404201040917.png)



通常把character_set_client，character_set_connection,character_set_results这三个系统变量设置成和客户端使用的字符集一致的情况，这样减少了很多无谓的字符集转换。为了方便我们设置，mysql提供了一条简单的语句

```mysql
SET NAMES 字符集名;
```

# SQL大小写规范

 

**Windows和Linux平台区别**

在 SQL 中，关键字和函数名是不用区分字母大小写的，比如 SELECT、WHERE、ORDER、GROUP BY 等关键字，以及 ABS、MOD、ROUND、MAX 等函数名。不过在 SQL 中，你还是要确定大小写的规范，

因为在 Linux 和 Windows 环境下，你可能会遇到不同的大小写问题。 **windows系统默认大小写不敏感** ，但是 **linux系统是大小写敏感的** 

通过如下命令查看:

```mysql
SHOW VARIABLES LIKE '%lower_case_table_names%'
```

* Windows系统如下8.0:

  ![image-20220404201808849](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220404201808849.png)

* linux8.0下:

  ![image-20220404201920919](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220404201920919.png)

* lower_case_table_names参数值的设置：

	*  默认为0，大小写敏感 。
	*  设置1，大小写不敏感。创建的表，数据库都是以小写形式存放在磁盘上，对于sql语句都是转换为小写对表和数据库进行查找。
	* 设置2，创建的表和数据库依据语句上格式存放，凡是查找都是转换为小写进行。

两个平台上SQL大小写的区别具体来说：

> MySQL在Linux下数据库名、表名、列名、别名大小写规则是这样的：
>
> 1、数据库名、表名、表的别名、变量名是严格区分大小写的；
>
> 2、关键字、函数名称在 SQL 中不区分大小写；
>
> 3、列名（或字段名）与列的别名（或字段别名）在所有的情况下均是忽略大小写的；
>
> **MySQL在Windows的环境下全部不区分大小写**

