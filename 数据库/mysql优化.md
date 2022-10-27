---
title: mysql优化
categories:
  - 数据库
tags:
  - mysql
abbrlink: 56288
date: 2022-3-12 19:54:20
cover : https://img.php.cn/upload/article/000/000/024/61505c2a73cf1419.jpg 
---



## 图解

![image-20220809221050360](../../images/mysql%E4%BC%98%E5%8C%96/image-20220809221050360.png)

# Explain详解

以下两节均来自诸葛老师笔记，仅作学习。

## Explain工具介绍

使用EXPLAIN关键字可以模拟优化器执行SQL语句，分析你的查询语句或是结构的性能瓶颈

在 select 语句之前增加 explain 关键字，MySQL 会在查询上设置一个标记，执行查询会返回执行计划的信息，而不是
执行这条SQL
注意：如果 from 中包含子查询，仍会执行该子查询，将结果放入临时表中

## Explain分析示例

参考官方文档：https://dev.mysql.com/doc/refman/5.7/en/explain-output.html


```mysql

1 示例表：
2 DROP TABLE IF EXISTS `actor`;
3 CREATE TABLE `actor` (
4 `id` int( 11 ) NOT NULL,
5 `name` varchar( 45 ) DEFAULT NULL,
6 `update_time` datetime DEFAULT NULL,
7 PRIMARY KEY (`id`)
8 ) ENGINE=InnoDB DEFAULT CHARSET=utf8;
9
10 INSERT INTO `actor` (`id`, `name`, `update_time`) VALUES ( 1 ,'a','2017‐12‐
15:27:18'), ( 2 ,'b','2017‐12‐22 15:27:18'), ( 3 ,'c','2017‐12‐22 15:27:18');
11
12 DROP TABLE IF EXISTS `film`;
13 CREATE TABLE `film` (
14 `id` int( 11 ) NOT NULL AUTO_INCREMENT,
15 `name` varchar( 10 ) DEFAULT NULL,
16 PRIMARY KEY (`id`),
17 KEY `idx_name` (`name`)
18 ) ENGINE=InnoDB DEFAULT CHARSET=utf8;
19
20 INSERT INTO `film` (`id`, `name`) VALUES ( 3 ,'film0'),( 1 ,'film1'),( 2 ,'film2');
21
22 DROP TABLE IF EXISTS `film_actor`;
23 CREATE TABLE `film_actor` (
24 `id` int( 11 ) NOT NULL,
25 `film_id` int( 11 ) NOT NULL,
26 `actor_id` int( 11 ) NOT NULL,
27 `remark` varchar( 255 ) DEFAULT NULL,
28 PRIMARY KEY (`id`),
29 KEY `idx_film_actor_id` (`film_id`,`actor_id`)
30 ) ENGINE=InnoDB DEFAULT CHARSET=utf8;
31
32 INSERT INTO `film_actor` (`id`, `film_id`, `actor_id`) VALUES ( 1 , 1 , 1 ),( 2 , 1 , 2 ),( 3 , 2 , 1 );


```



```mysql
1 mysql> explain select * from actor;
```

在查询中的每个表会输出一行，如果有两个表通过 join 连接查询，那么会输出两行

## explain中的列

接下来我们将展示 explain 中每个列的信息。

![image-20220302173512259](../../images/mysql%E6%B7%B1%E5%85%A5%E6%B5%85%E5%87%BA%E7%B4%A2%E5%BC%95/image-20220302173512259.png)

**1.id列**

id列的编号是 select 的序列号，有几个 select 就有几个id，并且id的顺序是按 select 出现的顺序增长的。

id列越大执行优先级越高，id相同则从上往下执行，id为NULL最后执行。

**2.select_type列**
select_type 表示对应行是简单还是复杂的查询。

- ​        **simple**：简单查询。查询不包含子查询和union

  ```mysql
         mysql> explain select * from film where id = 2 ;
  ```

- ​        **primary**：复杂查询中最外层的 select

- ​        **subquery**：包含在 select 中的子查询（不在 from 子句中）

- ​        **derived**：包含在 from 子句中的子查询。MySQL会将结果存放在一个临时表中，也称为派生表（derived的英文含义）



**3.table列**
这一列表示 explain 的一行正在访问哪个表。

当 from 子句中有子查询时，table列是 格式，表示当前查询依赖 id=N 的查询，于是先执行 id=N 的查询。

当有 union 时，UNION RESULT 的 table 列的值为<union1,2>，1和2表示参与 union 的 select 行id。

**4.type列(重要)**

这一列表示关联类型或访问类型，即MySQL决定如何查找表中的行，查找数据行记录的大概范围。

**依次从最优到最差分别为：system > const > eq_ref > ref > range > index > ALL**

一般来说，得保证查询达到range级别，最好达到ref

> ​    4.1 NULL：mysql能够在优化阶段分解查询语句，在执行阶段用不着再访问表或索引。例如：在索引列中选取最小值，可
>
> ​    以单独查找索引来完成，不需要在执行时访问表
>
> ```mysql
>     mysql> explain select min(id) from film;
> ```
>
> 
>
> ​     4.2 const, system：mysql能对查询的某部分进行优化并将其转化成一个常量（可以看show warnings 的结果）。用于
> ​    primary key 或 unique key 的所有列与常数比较时，所以表最多有一个匹配行，读取1次，速度比较快。system是
> ​    const的特例，表里只有一条元组匹配时为system
> mysql> explain extended select * from (select * from film where id = 1 ) tmp;
>
> ```
> 1 mysql> show warnings;
> ```
>
> ​      4.3    eq_ref：primary key 或 unique key 索引的所有部分被连接使用 ，最多只会返回一条符合条件的记录。这可能是在
> ​      const 之外最好的联接类型了，简单的 select 查询不会出现这种 type。
> ​     mysql> explain select * from film_actor left join film on film_actor.film_id = film.id;
>
> ​     4.4    ref：相比 eq_ref，不使用唯一索引，而是使用普通索引或者唯一性索引的部分前缀，索引要和某个值相比较，可能会
> 找到多个符合条件的行。
>
> ​          1.简单 select 查询，name是普通索引（非唯一索引）
>
> ```mysql
>          mysql> explain select * from film where name = 'film1';
> ```
>
> 
>
> ​        2.关联表查询，idx_film_actor_id是film_id和actor_id的联合索引，这里使用到了film_actor的左边前缀film_id部分。
>
> ```mysql
>          mysql> explain select film_id from film left join film_actor on film.id = film_actor.film_id;
> ```
>
> ​     4.5   range：范围扫描通常出现在 in(), between ,> ,<, >= 等操作中。使用一个索引来检索给定范围的行。
> ​     mysql> explain select * from actor where id > 1 ;
>
> ​     4.6    index：扫描全索引就能拿到结果，一般是扫描某个二级索引，这种扫描不会从索引树根节点开始快速查找，而是直接
> ​    对二级索引的叶子节点遍历和扫描，速度还是比较慢的，这种查询一般为使用覆盖索引，二级索引一般比较小，所以这
> ​    种通常比ALL快一些。
> ​      mysql> explain select * from film;
>
> ​     4.7    ALL：即全表扫描，扫描你的聚簇索引的所有叶子节点。通常情况下这需要增加索引来进行优化了。
>
> ​            
>
> ```mysql
> mysql> explain select * from actor;
> ```
>
> 



**5.possible_keys列**
这一列显示查询可能使用哪些索引来查找。
explain 时可能出现 possible_keys 有列，而 key 显示 NULL 的情况，这种情况是因为表中数据不多，mysql认为索引
对此查询帮助不大，选择了全表查询。
如果该列是NULL，则没有相关的索引。在这种情况下，可以通过检查 where 子句看是否可以创造一个适当的索引来提
高查询性能，然后用 explain 查看效果。



**6.key列**
这一列显示mysql实际采用哪个索引来优化对该表的访问。
如果没有使用索引，则该列是 NULL。如果想强制mysql使用或忽视possible_keys列中的索引，在查询中使用 force
index、ignore index。

**7.key_len列**

这一列显示了mysql在索引里使用的字节数，通过这个值可以算出具体使用了索引中的哪些列。
举例来说，film_actor的联合索引 idx_film_actor_id 由 film_id 和 actor_id 两个int列组成，并且每个int是4字节。通
过结果中的key_len=4可推断出查询使用了第一个列：film_id列来执行索引查找。

```mysql
mysql> explain select * from film_actor where film_id = 2 ;
```

key_len计算规则如下：
字符串，char(n)和varchar(n)，5.0.3以后版本中，n均代表字符数，而不是字节数，如果是utf-8，一个数字
或字母占1个字节，一个汉字占3个字节

char(n)：如果存汉字长度就是 3n 字节

varchar(n)：如果存汉字则长度是 3n + 2 字节，加的2字节用来存储字符串长度，因为

varchar是变长字符串

数值类型

tinyint：1字节

smallint：2字节

int：4字节

bigint：8字节

时间类型

date：3字节

timestamp：4字节

datetime：8字节

如果字段允许为 NULL，需要1字节记录是否为 NULL

索引最大长度是768字节，当字符串过长时，mysql会做一个类似左前缀索引的处理，将前半部分的字符提取出来做索
引。

**8.ref列**

这一列显示了在key列记录的索引中，表查找值所用到的列或常量，常见的有：const（常量），字段名（例：film.id）

**9.rows列**
这一列是mysql估计要读取并检测的行数，注意这个不是结果集里的行数。

**10.Extra列**
这一列展示的是额外信息。常见的重要值如下：

1）**Using index**：使用覆盖索引
覆盖索引定义：mysql执行计划explain结果里的key有使用索引，如果select后面查询的字段都可以从这个索引的树中
获取，这种情况一般可以说是用到了覆盖索引，extra里一般都有using index；覆盖索引一般针对的是辅助索引，整个
查询结果只通过辅助索引就能拿到结果，不需要通过辅助索引树找到主键，再通过主键去主键索引树里获取其它字段值

  mysql> explain select film_id from film_actor where film_id = 1 ;

2）**Using where**：使用 where 语句来处理结果，并且查询的列未被索引覆盖

  mysql> explain select * from actor where name = 'a';

3）**Using index condition**：查询的列不完全被索引覆盖，where条件中是一个前导列的范围；
 mysql> explain select * from film_actor where film_id > 1 ;

4）Using temporary：mysql需要创建一张临时表来处理查询。出现这种情况一般是要进行优化的，首先是想到用索
引来优化。

1. actor.name没有索引，此时创建了张临时表来distinct
   1 mysql> explain select distinct name from actor;
2. film.name建立了idx_name索引，此时查询时extra是using index,没有用临时表
   1 mysql> explain select distinct name from film;

5）**Using filesort**：将用外部排序而不是索引排序，数据较小时从内存排序，否则需要在磁盘完成排序。这种情况下一
般也是要考虑使用索引来优化的。

1. actor.name未创建索引，会浏览actor整个表，保存排序关键字name和对应的id，然后排序name并检索行记录
   mysql> explain select * from actor order by name;

1. film.name建立了idx_name索引,此时查询时extra是using index
   mysql> explain select * from film order by name;

6）Select tables optimized away：使用某些聚合函数（比如 max、min）来访问存在索引的某个字段是
        mysql> explain select min(id) from film;





# 普通索引与唯一索引怎样选择



## 查询：

普通索引找到主键值然后回表查询真实记录，然后接着向后根据条件查询记录。

唯一索引找到主键值然后回表查询真实记录后，查找的过程就会停止

在查询过程中，这两种索引情况差不多，效率差不多

## 更新：

普通索引的更新会使用到change buffer；

唯一索引的更新不会使用到cahnge buffer；



**change buffer**

change buffer是内存中的一块区域，它保存在innodb的buffer pool中，占到innodb buffer pool的25%大小。



**change buffer使用**



当我们需要更新一条数据的时候，如果这条记录所在的数据页A在内存中，那么直接更新内存中的数据页A既可，如果数据页不在内存中，这个时候，innodb会将这些更新操作缓存在change buffer中，当下次需要访问磁盘上的数据页B时，将数据页B从磁盘上加载到内存里，然后应用change buffer中与这个数据页有关的操作。



**使用change buffer的好处**



- 将原来2次的磁盘访问，整合成一次磁盘访问，并且能够保证数据的一致性。
- 数据页B读入内存是需要占用内存空间的，这种方式能避免内存的使用，提高内存的利用率



**为什么唯一索引不能使用change buffer**

原因是唯一索引在做insert或者update时，需要判断索引记录的唯一性，而判断唯一性必须要在内存中判断，所以数据页会别加载到内存中，如果数据页已经加载到了内存中，那么当然是直接更新内存更快了。



## 普通索引与唯一索引更新时的差异

以update table where k=4举例子：

**1.当要更新的记录在内存中的时候**

普通索引k会找到索引记录等于3和5的位置，然后再中间插入k = 4的记录。

唯一索引k会找到索引记录等于3和5的位置，然后判断没有冲突，然后插入k = 4的记录

当记录在内存中的时候，这两差别不大，仅仅是唯一索引多了一步判断唯一性的操作，这个判断不太影响。

**2.当要更新的记录不在内存的时候**

唯一索引要将数据页加载到内存中，判断这个值有没有冲突，然后插入这个值

普通索引则是将更新记录在change buffer，语句执行就结束了

可以看到，普通索引利用了change buffer,减少了磁盘上的随机访问，对性能的提升比较明显



**总结：查询都可以，修改数据建议普通索引**



**redo log 主要节省的是随机写磁盘的 IO 消耗（转成顺序写），而 change buffer 主要节省的则是随机读磁盘的 IO 消耗。**



# 怎么给字符串字段加索引



**使用前缀索引，定义好长度，就可以做到既节省空间，又不用额外增加太多的查询成本。**



实际上，我们在建立索引时关注的是区分度，区分度越高越好。因为区分度越高，意味着重复的键值越少。因此，我们可以通过统计索引上有多少个不同的值来判断要使用多长的前缀。



# WAL机制

**写日志，再写磁盘。**

WAL，全称是Write-Ahead Logging， 预写日志系统。指的是 MySQL 的写操作并不是立刻更新到磁盘上，而是先记录在日志上，然后在合适的时间再更新到磁盘上。这样的好处是错开高峰期。日志主要分为 undo log、redo log、binlog。这三种在之前的博客已经详细说过了，作用分别是 " 完成MVCC从而实现 MySQL 的隔离级别 "、" 降低随机写的性能消耗（转成顺序写），同时防止写操作因为宕机而丢失 "、" 写操作的备份，保证主从一致 "。关于这三种日志的内容讲的比较分散且具体的执行过程没有提到，所以这里来总结一下这三种日志。



`mysql` 每执行一条 `DML` 语句，先将记录写入 `redo log buffer`，后续某个时间点再一次性将多个操作记录写到 `redo log file`。这种 **先写日志，再写磁盘** 的技术就是 `MySQL` 里经常说到的 `WAL(Write-Ahead Logging)` 技术。

# redo log 与 undo log 与bin log

## redolog与binlog的区别



**适用对象不同**：binlog是Mysql的server层实现的日志，所有存储引擎都可以适用。redlog是Innodb存储引擎特有的日志

**文件格式不同**：redolog记录了更新数据的操作对数据页的修改。binlog有三种实现方式，大致是记录sql语句，以二进制的形式存放在文件中。

**写入方式不同**：redolog是循环写，日志空间大小固定，全部写满就从头开始，会覆盖之前的日志数据。binlog是追加写，写满一个文件，就创建一个新的文件继续写，不会覆盖之前写的数据。

**用途不同**：redolog多用在掉电等故障恢复，binlog用于备份恢复，主从复制。







## undo log 

回滚日志



undo log 主要用于实现 MVCC，从而实现 MySQL 的 ”读已提交“、”可重复读“ 隔离级别。在每个行记录后面有两个隐藏列，"trx_id"、"roll_pointer"，分别表示上一次修改的事务id，以及 "上一次修改之前保存在 undo log中的记录位置 "。在对一行记录进行修改或删除操作前，会先将该记录拷贝一份到 undo log 中，然后再进行修改，并将修改事务 id，拷贝的记录在 undo log 中的位置写入 "trx_id"、"roll_pointer"。



## Redo log



redo log 是搭配缓冲池、change buffer 使用的，缓冲池的作用是缓存磁盘上的数据页，减少磁盘的IO；change buffer 的作用是将写操作先存在内存中，等到下次需要读取这些操作涉及到的数据页时，就把数据页加载到缓冲池中，然后在缓冲池中更新；redo log 的作用就是持久化记录的写操作，防止在写操作更新到磁盘前发生断电丢失这些写操作，**直到该操作对应的脏页真正落盘（先读取数据页到缓冲池然后应用写操作到缓冲池，最后再将脏页落盘替换磁盘上的数据页），该操作才会从 redo log 中移除（覆盖）**。

redolog记录的是**写操作对数据页的修改逻辑**以及 change buffer的变化。



![image-20220312163150544](../../images/mysql%E4%BC%98%E5%8C%96/image-20220312163150544.png)





### redolog写入机制

这三种状态分别是：

1. 存在redo log buffer中，物理上是在MySQL进程内存中，就是图中的红色部分；

2. 写到磁盘(write)，但是没有持久化（fsync)，物理上是在文件系统的page cache里面，也就是图中的黄色部分；

3. 持久化到磁盘，对应的是hard disk，也就是图中的绿色部分。

   

**InnoDB有一个后台线程，每隔1秒，就会把redo log buffer中的日志，调用write写到文件系统的page cache，然后调用fsync持久化到磁盘。**





**`redo log` 它是物理日志**

redo log 是物理日志，记录了某个数据页做了什·么修改，对 XXX 表空间中的 YYY 数据页 ZZZ 偏移量的地方做了AAA 更新，每当执行一个事务就会产生这样的一条或者多条物理日志。



## Binlog

binlog 也是保存写操作的，但是它主要是用于进行集群中保证主从一致以及执行异常操作后恢复数据的。

![image-20220809220930307](../../images/mysql%E4%BC%98%E5%8C%96/image-20220809220930307.png)





先写日志，再写磁盘，但是更新内存数据是再写redo日志前。



**也就是，一条更新sql先将旧值写进undo日志，再更新内存中的缓存，再在redo log buffer写redo日志，然后将redo日志刷入磁盘（此时redo log处于prepare阶段,这个刷盘是后台线程每隔一秒刷的），再写binlog日志进磁盘然后，最后提交事务处于commit状态（把写进磁盘的redo log日志改成commit状态）。**





**binlog的写入逻辑比较简单：事务执行过程中，先把日志写到binlog cache，事务提交的时候，再把binlog cache写到binlog文件中。**

1. 先把binlog从binlog cache中写到磁盘上的binlog文件；
2. 调用fsync持久化。







**`binlog` 是逻辑日志，记录内容是语句的原始逻辑**



**以二进制的形式存放到文件中**





有三种工作模式

- 每一条会修改数据的sql都会记录在binlog中。
- 不记录sql语句上下文相关信息，仅保存哪条记录被修改。
- 以上两种level的混合使用，一般的语句修改使用statment格式保存binlog









# mysql优化

## Order By与Group by优化

  

### Order BY 

explain的Extra字段如果是Using filesort则代表排序没有走索引，如果是Using index则是走了索引。

**联合字段name age position**

**case1**

![image-20220312200151076](../../images/mysql%E4%BC%98%E5%8C%96/image-20220312200151076.png)![image-20220312200151170](../../images/mysql%E4%BC%98%E5%8C%96/image-20220312200151170.png)



**case2**

查找只用到索引name，age和position用于排序，无Using filesort

![image-20220312200310387](../../images/mysql%E4%BC%98%E5%8C%96/image-20220312200310387.png)



explain的执行结果一样，但是出现了Using filesort，因为索引的创建顺序为name,age,position，但是排序的时候age和position颠倒位置了,所以不能走所以排序，走了文件排序。

**case3**

![image-20220312200808675](../../images/mysql%E4%BC%98%E5%8C%96/image-20220312200808675.png)

这种情况下，不能走索引排序，因为图中这种情况，根据name筛选的记录中，第一个字段是不一样的，如果按照age 和position排序的话必须得第一个字段相同。

 **优化总结：**

1、MySQL支持两种方式的排序filesort和index，Using index是指MySQL扫描索引本身完成排序。index效率高，filesort效率低。

2、order by满足两种情况会使用Using index。

- ​           order by语句使用索引最左前列。
- ​           使用where子句与   by子句条件列组合满足索引最左前列。

3、尽量在索引列上完成排序，遵循索引建立（索引创建的顺序）时的最左前缀法则。

4、如果order by的条件不在索引列上，就会产生Using filesort。

5、能用覆盖索引尽量用覆盖索引

6、group by与order by很类似，其实质是先排序后分组，遵照索引创建顺序的最左前缀法则。对于group by的优化如果不需要排序的可以加上**order by null禁止排序**。注意，where高于having，能写在where中的限定条件就不要去having限定了。



### **Using filesort文件排序原理详解**

**filesort文件排序方式**

- 单路排序：是一次性取出满足条件行的所有字段，然后在sort buffer中进行排序；用trace工具可以看到sort_mode信息里显示< sort_key, additional_fields >或者< sort_key, packed_additional_fields >
- 双路排序（又叫**回表**排序模式）：是首先根据相应的条件取出相应的**排序字段**和**可以直接定位行数据的行 ID**，然后在 sort buffer 中进行排序，排序完后需要再次取回其它需要的字段；用trace工具可以看到sort_mode信息里显示< sort_key, rowid >

MySQL 通过比较系统变量 max_length_for_sort_data(**默认1024字节**) 的大小和需要查询的字段总大小来判断使用哪种排序模式。

- 如果 字段的总长度小于max_length_for_sort_data ，那么使用 单路排序模式；
- 如果 字段的总长度大于max_length_for_sort_data ，那么使用 双路排序模·式。



 我们先看**单路排序**的详细过程：

1. 从索引name找到第一个满足 name = ‘zhuge’ 条件的主键 id
2. 根据主键 id 取出整行，**取出所有字段的值，存入 sort_buffer 中**
3. 从索引name找到下一个满足 name = ‘zhuge’ 条件的主键 id
4. 重复步骤 2、3 直到不满足 name = ‘zhuge’ 
5. 对 sort_buffer 中的数据按照字段 position 进行排序
6. 返回结果给客户端

我们再看下**双路排序**的详细过程：

1. 从索引 name 找到第一个满足 name = ‘zhuge’  的主键id
2. 根据主键 id 取出整行，**把排序字段 position 和主键 id 这两个字段放到 sort buffer 中**
3. 从索引 name 取下一个满足 name = ‘zhuge’  记录的主键 id
4. 重复 3、4 直到不满足 name = ‘zhuge’ 
5. 对 sort_buffer 中的字段 position 和主键 id 按照字段 position 进行排序
6. 遍历排序好的 id 和字段 position，按照 id 的值**回到原表**中取出 所有字段的值返回给客户端

其实对比两个排序模式，单路排序会把所有需要查询的字段都放到 sort buffer 中，而双路排序只会把主键和需要排序的字段放到 sort buffer 中进行排序，然后再通过主键回到原表查询需要的字段。

如果 MySQL **排序内存** **sort_buffer** 配置的比较小并且没有条件继续增加了，可以适当把 max_length_for_sort_data 配置小点，让优化器选择使用**双路排序**算法，可以在sort_buffer 中一次排序更多的行，只是需要再根据主键回到原表取数据。

如果 MySQL 排序内存有条件可以配置比较大，可以适当增大 max_length_for_sort_data 的值，让优化器优先选择全字段排序(**单路排序**)，把需要的字段放到 sort_buffer 中，这样排序后就会直接从内存里返回查询结果了。

所以，MySQL通过 **max_length_for_sort_data** 这个参数来控制排序，在不同场景使用不同的排序模式，从而提升排序效率。



## limit优化

很多时候我们业务系统实现分页功能可能会用如下sql实现

​     

```mysql
mysql> select * from employees limit 10000,10;            
```

  

表示从表 employees 中取出从 10001 行开始的 10 行记录。看似只查询了 10 条记录，实际这条 SQL 是先读取 10010 条记录，然后抛弃前 10000 条记录，然后读到后面 10 条想要的数据。因此要查询一张大表比较靠后的数据，执行效率是非常低的。



**>>常见的分页场景优化技巧：**



### **1、根据自增且连续的主键排序的分页查询**



​                

```mysql
select * from employees limit 90000,5;//这样写，之前的90000条数据也会扫描，可就太慢了             
```

​    这条sql就可以优化为

```mysql
select * from employees where id > 90000 limit 5;              
```



但这种情况很罕见，而且局限性很大，比如上面优化后的sql,一旦你删了90001记录，那两条结果就不一样了







如果查找结果集太大可能就不走索引走全表，因为要不断回表，效率太低。

平常我们的数据不一定连读而且不一定是用主键排序，而是用非主键字段排序进行分页查询.

### **2、根据非主键字段排序的分页查询**

再看一个根据非主键字段排序的分页查询，SQL 如下：

```mysql
mysql>  select * from employees ORDER BY name limit 90000,5;
```



![img](../../images/mysql%E4%BC%98%E5%8C%96/100107)



发现并没有使用 name 字段的索引（key 字段对应的值为 null），具体原因：**扫描整个索引并查找到没索引的行(可能要遍历多个索引树)的成本比扫描全表的成本更高，所以优化器放弃使用索引,走全表扫描效率更高**。









**那怎样优化？**

其实关键是**让排序时返回的字段尽可能少**，所以可以让排序和分页操作先查出主键，然后根据主键查到对应的记录，SQL改写如下：

​         

```mysql
select * from employees e inner join (select id from employees order by name limit 90000,5) ed on e.id = ed.id;              
```



需要的结果与原 SQL 一致，执行时间减少了一半以上，我们再对比优化前后sql的执行计划



**![https://note.youdao.com/yws/public/resource/df15aba3aa76c225090d04d0dc776dd9/xmlnote/2AD92E45574045DC8A95FEBD64DFFF49/100104](../../images/mysql%E4%BC%98%E5%8C%96/100104)**







原 SQL 使用的是 filesort 排序，而优化后的 SQL 使用的是索引排序。



### 优化

 **自己理解**：按照之前的limit 10000 10;把前10010条记录查询出来，这么多记录，优化器肯定不走索引，而是走全表扫描（性价比不高）；然后呢，怎样优化，是先排序，然后把那些分页的页的记录的主键id查询出来（就是10001到10010），然后生成临时表与这张表进行连接查询，为什么呢？首先，按照上面的那条括号sql，肯定是走索引的，因为查询字段只有主键，甚至不用回表，而且生成的临时表的记录肯定少，关联查询的话，就会走索引查询记录。







##  **Join关联查询优化**



**mysql的表关联常见有两种算法**

- Nested-Loop Join 算法

- Block Nested-Loop Join 算法



> ## NLJ
>
> **嵌套循环连接 Nested-Loop Join 算法**
>
> 
>
> 一次一行循环地从第一张表（称为**驱动表**）中读取行，在这行数据中取到关联字段，根据关联字段在另一张表（**被驱动表**）里取出满足条件的行，然后取出两张表的结果合集
>
> ```mysql
>              mysql> EXPLAIN select * from t1 inner join t2 on t1.a= t2.a;              
> ```
>
> 
>
> ![https://note.youdao.com/yws/public/resource/df15aba3aa76c225090d04d0dc776dd9/xmlnote/086C5EE10BB44DB5BC6AF3D21F97B75A/100112](../../images/mysql%E4%BC%98%E5%8C%96/100112)
>
> 
>
> 从执行计划中可以看到这些信息：
>
> - 驱动表是 t2，被驱动表是 t1。先执行的就是驱动表(执行计划结果的id如果一样则按从上到下顺序执行sql)；优化器一般会优先选择**小表做驱动表，**用where条件过滤完驱动表，然后再跟被驱动表做关联查询。**所以使用 inner join 时，排在前面的表并不一定就是驱动表。**
> - 当使用left join时，左表是驱动表，右表是被驱动表，当使用right join时，右表时驱动表，左表是被驱动表，当使用join时，mysql会选择数据量比较小的表作为驱动表，大表作为被驱动表。
> - 使用了 NLJ算法。一般 join 语句中，如果执行计划 Extra 中未出现 **Using join buffer** 则表示使用的 join 算法是 NLJ。
>
> **上面sql的大致流程如下：**
>
> 1. 从表 t2 中读取一行数据（如果t2表有查询过滤条件的，用先用条件过滤完，再从过滤结果里取出一行数据）；
> 2. 从第 1 步的数据中，取出关联字段 a，到表 t1 中查找；
> 3. 取出表 t1 中满足条件的行，跟 t2 中获取到的结果合并，作为结果返回给客户端；
> 4. 重复上面 3 步。
>
> 整个过程会读取 t2 表的所有数据(**扫描100行**)，然后遍历这每行数据中字段 a 的值，根据 t2 表中 a 的值索引扫描 t1 表中的对应行(**扫描100次 t1 表的索引，1次扫描可以认为最终只扫描 t1 表一行完整数据，也就是总共 t1 表也扫描了100行**)。因此整个过程扫描了 **200 行**。
>
> **如果被驱动表的关联字段没索引**，**使用NLJ算法性能会比较低(下面有详细解释)**，mysql会选择Block Nested-Loop Join算法。
>
> 
>
> ##  BNL
>
> **基于块的嵌套循环连接 Block Nested-Loop Join(BNL)算法**
>
> 
>
> 把**驱动表**的数据读入到 join_buffer 中，然后扫描**被驱动表**，把**被驱动表**每一行取出来跟 join_buffer 中的数据做对比。
>
> 
>
> ```mysql
> EXPLAIN select * from t1 inner join t2 on t1.b= t2.b;   
> ```
>
>    ![img](../../images/mysql%E4%BC%98%E5%8C%96/100111)
>
> ​        **上面的sql的执行流程：**
>
> 1. 把 t2 的所有数据放入到 **join_buffer** 中
> 2. 把表 t1 中每一行取出来，跟 join_buffer 中的数据做对比
> 3. 返回满足 join 条件的数据
>
> 整个过程对表 t1 和 t2 都做了一次全表扫描，因此扫描的总行数为10000(表 t1 的数据总量) + 100(表 t2 的数据总量) = **10100**。并且 join_buffer 里的数据是无序的，因此对表 t1 中的每一行，都要做 100 次判断，所以内存中的判断次数是 100 * 10000= **100 万次**。
>
> 这个例子里表 t2 才 100 行，要是表 t2 是一个大表，join_buffer 放不下怎么办呢？·
>
> join_buffer 的大小是由参数 join_buffer_size 设定的，默认值是 256k。如果放不下表 t2 的所有数据话，策略很简单，就是**分段放**。
>
> 比如 t2 表有1000行记录， join_buffer 一次只能放800行数据，那么执行过程就是先往 join_buffer 里放800行记录，然后从 t1 表里取数据跟 join_buffer 中数据对比得到部分结果，然后清空  join_buffer ，再放入 t2 表剩余200行记录，再次从 t1 表里取数据跟 join_buffer 中数据对比。所以就多扫了一次 t1 表。





### 总结

**被驱动表的关联字段没索引为什么要选择使用 BNL 算法而不使用 Nested-Loop Join 呢？**

如果上面第二条sql使用 Nested-Loop Join，那么扫描行数为 100 * 10000 = 100万次，这个是**磁盘扫描**。

很显然，用BNL磁盘扫描次数少很多，相比于磁盘扫描，BNL的内存计算会快得多。

因此MySQL对于被驱动表的关联字段没索引的关联查询，一般都会使用 BNL 算法。如果有索引一般选择 NLJ 算法，有索引的情况下 NLJ 算法比 BNL算法性能更高

而如果使用NLJ算法，驱动表t2加载到**join_buffer**，然后把被驱动表t1的每一行取出来与join_buffer对比，扫描的总行数是10100,比100万少很多。扫**描次数是两表记录之和。**



**被驱动表的关联字段索引为什么要选择使用 NLJ算法**

在驱动表t2一条一条读取行记录，然后根据关联字段去被驱动表t2走索引去取符合条件的记录，扫描磁盘的次数只有200行。**扫描记录是小表记录之和*2；**



> 第一个问题：能不能使用join语句？
>
> 1. 如果可以使用Index Nested-Loop Join算法，也就是说可以用上被驱动表上的索引，其实是没问题的；
> 2. 如果使用Block Nested-Loop Join算法，扫描行数就会过多。尤其是在大表上的join操作，这样可能要扫描被驱动表很多次，会占用大量的系统资源。所以这种join尽量不要用。
>
> 所以你在判断要不要使用join语句时，就是看explain结果里面，Extra字段里面有没有出现“Block Nested Loop”字样。
>
> 第二个问题是：如果要使用join，应该选择大表做驱动表还是选择小表做驱动表？
>
> 1. 如果是Index Nested-Loop Join算法，应该选择小表做驱动表；
> 2. 如果是Block Nested-Loop Join算法：
>    - 在join_buffer_size足够大的时候，是一样的；
>    - 在join_buffer_size不够大的时候（这种情况更常见），应该选择小表做驱动表。
>
> 所以，这个问题的结论就是，总是应该使用小表做驱动表。





**对于小表定义的明确**

**在决定哪个表做驱动表的时候，应该是两个表按照各自的条件过滤，过滤完成之后，计算参与join的各个字段的总数据量，数据量小的那个表，就是“小表”，应该作为驱动表。**



### 优化

**对于关联sql的优化**

- 关联字段加索引，让mysql做join操作时尽量选择NLJ算法，驱动表因为需要全部查询出来，所以过滤的条件也尽量走索引，避免全表扫描，总之，能走索引的过滤条件尽量都走索引
- 小表驱动大表，写多表连接sql时如果**明确知道**哪张表是小表可以用straight_join写法固定连接驱动方式，省去mysql优化器自己判断的时间





## **in和exsits优化**

原则：**小表驱动大表**，即小的数据集驱动大的数据集





**in：**当B表的数据集小于A表的数据集时，in优于exists 



```mysql
select * from A where id in (select id from B)   

#等价于： 　

for(select id from B){     

select * from A where A.id = B.id   

 }              
```



**exists：**当A表的数据集小于B表的数据集时，exists优于in

　　将主查询A的数据，放到子查询B中做条件验证，根据验证结果（true或false）来决定主查询的数据是否保留

```mysql
 select * from A where exists (select 1 from B where B.id = A.id) 

#等价于:    for(select * from A){      

select * from B where B.id = A.id  

  }   
 #A表与B表的ID字段应建立索引              


```





## **count(\*)查询优化**

​           

```mysql
mysql> EXPLAIN select count(1) from employees; 

mysql> EXPLAIN select count(id) from employees;

 mysql> EXPLAIN select count(name) from employees;

 mysql> EXPLAIN select count(*) from employees;              
```



> **四个sql的执行计划一样，说明这四个sql执行效率应该差不多**
>
> **字段有索引：count(\*)≈count(1)>count(字段)>count(主键 id)    //字段有索引，count(字段)统计走二级索引，二级索引存储数据比主键索引少，所以count(字段)>count(主键 id)** 
>
> **字段无索引：count(\*)≈count(1)>count(主键 id)>count(字段)    //字段没有索引count(字段)统计走不了索引，count(主键 id)还可以走主键索引，所以count(主键 id)>count(字段)**
>
> 
>
> count(1)跟count(字段)执行过程类似，不过count(1)不需要取出字段统计，就用常量1做统计，count(字段)还需要取出字段，所以理论上count(1)比count(字段)会快一点。
>
> count(*) 是例外，mysql并不会把全部字段取出来，而是专门做了优化，不取值，按行累加，效率很高，所以不需要用count(列名)或count(常量)来替代 count(*)。
>
> 为什么对于count(id)，mysql最终选择辅助索引而不是主键聚集索引？因为二级索引相对主键索引存储数据更少，检索性能应该更高，mysql内部做了点优化(应该是在5.7版本才优化)。
>
> **常见优化方法**
>
> **1、查询mysql自己维护的总行数**
>
> 对于**myisam存储引擎**的表做不带where条件的count查询性能是很高的，因为myisam存储引擎的表的总行数会被mysql存储在磁盘上，查询不需要计算
>
> ​    ![0](../../images/mysql%E4%BC%98%E5%8C%96/100114)
>
> 对于**innodb存储引擎**的表mysql不会存储表的总记录行数(因为有MVCC机制，后面会讲)，查询count需要实时计算
>
> **2、show table status**
>
> 如果只需要知道表总行数的估计值可以用如下sql查询，性能很高
>
> ​    ![0](../../images/mysql%E4%BC%98%E5%8C%96/100115)
>
> **3、将总数维护到Redis里**
>
> 插入或删除表数据行的时候同时维护redis里的表总行数key的计数值(用incr或decr命令)，但是这种方式可能不准，很难保证表操作和redis操作的事务一致性
>
> **4、增加数据库计数表**
>
> 插入或删除表数据行的时候同时维护计数表，让他们在同一个事务里操作





**那为什么 InnoDB 不跟 MyISAM 一样，也把数字存起来呢？**

大家都知道MySAM在slect count()的时候，效率很高，是因为其内部会维护一个count值，查的时候直接返回那个维护的count值就可以了，那Innodb为什么 不和MySAM一样维护一个值呢，这是因为即使是在同一个时刻的多个查询，由于多版本并发控制（MVCC）的原因，InnoDB 表“应该返回多少行”也是不确定的。

# **索引设计原则与实战**

## 索引设计原则

**1、代码先行，索引后上**

不知大家一般是怎么给数据表建立索引的，是建完表马上就建立索引吗？

这其实是不对的，一般应该等到主体业务功能开发完毕，把涉及到该表相关sql都要拿出来分析之后再建立索引。

**2、联合索引尽量覆盖条件**

比如可以设计一个或者两三个联合索引(尽量少建单值索引)，让每一个联合索引都尽量去包含sql语句里的where、order by、group by的字段，还要确保这些联合索引的字段顺序尽量满足sql查询的最左前缀原则。

**3、不要在小基数字段上建立索引**

索引基数是指这个字段在表里总共有多少个不同的值，比如一张表总共100万行记录，其中有个性别字段，其值不是男就是女，那么该字段的基数就是2。

如果对这种小基数字段建立索引的话，还不如全表扫描了，因为你的索引树里就包含男和女两种值，根本没法进行快速的二分查找，那用索引就没有太大的意义了。

一般建立索引，尽量使用那些基数比较大的字段，就是值比较多的字段，那么才能发挥出B+树快速二分查找的优势来。

**4、长字符串我们可以采用前缀索引**

尽量对字段类型较小的列设计索引，比如说什么tinyint之类的，因为字段类型较小的话，占用磁盘空间也会比较小，此时你在搜索的时候性能也会比较好一点。

当然，这个所谓的字段类型小一点的列，也不是绝对的，很多时候你就是要针对varchar(255)这种字段建立索引，哪怕多占用一些磁盘空间也是有必要的。

对于这种varchar(255)的大字段可能会比较占用磁盘空间，可以稍微优化下，比如针对这个字段的前20个字符建立索引，就是说，对这个字段里的每个值的前20个字符放在索引树里，类似于 KEY index(name(20),age,position)。

此时你在where条件里搜索的时候，如果是根据name字段来搜索，那么此时就会先到索引树里根据name字段的前20个字符去搜索，定位到之后前20个字符的前缀匹配的部分数据之后，再回到聚簇索引提取出来完整的name字段值进行比对。

但是假如你要是order by name，那么此时你的name因为在索引树里仅仅包含了前20个字符，所以这个排序是没法用上索引的， group by也是同理。所以这里大家要对前缀索引有一个了解。

**5、where与order by冲突时优先where**

在where和order by出现索引设计冲突时，到底是针对where去设计索引，还是针对order by设计索引？到底是让where去用上索引，还是让order by用上索引?

一般这种时候往往都是让where条件去使用索引来快速筛选出来一部分指定的数据，接着再进行排序。

因为大多数情况基于索引进行where筛选往往可以最快速度筛选出你要的少部分数据，然后做排序的成本可能会小很多。

**6、基于慢sql查询做优化**

可以根据监控后台的一些慢sql，针对这些慢sql查询做特定的索引优化。

关于慢sql查询不清楚的可以参考这篇文章：https://blog.csdn.net/qq_40884473/article/details/89455740



## **索引设计实战**

以社交场景APP来举例，我们一般会去搜索一些好友，这里面就涉及到对用户信息的筛选，这里肯定就是对用户user表搜索了，这个表一般来说数据量会比较大，我们先不考虑分库分表的情况，比如，我们一般会筛选地区(省市)，性别，年龄，身高，爱好之类的，有的APP可能用户还有评分，比如用户的受欢迎程度评分，我们可能还会根据评分来排序等等。

对于后台程序来说除了过滤用户的各种条件，还需要分页之类的处理，可能会生成类似sql语句执行：

select xx from user where xx=xx and xx=xx order by xx limit xx,xx

对于这种情况如何合理设计索引了，比如用户可能经常会根据省市优先筛选同城的用户，还有根据性别去筛选，那我们是否应该设计一个联合索引 (province,city,sex) 了？这些字段好像基数都不大，其实是应该的，因为这些字段查询太频繁了。

假设又有用户根据年龄范围去筛选了，比如 where  province=xx and city=xx and age>=xx and age<=xx，我们尝试着把age字段加入联合索引 (province,city,sex,age)，注意，一般这种范围查找的条件都要放在最后，之前讲过联合索引范围之后条件的是不能用索引的，但是对于当前这种情况依然用不到age这个索引字段，因为用户没有筛选sex字段，那怎么优化了？其实我们可以这么来优化下sql的写法：where  province=xx and city=xx and sex in ('female','male') and age>=xx and age<=xx

对于爱好之类的字段也可以类似sex字段处理，所以可以把爱好字段也加入索引 (province,city,sex,hobby,age) 

假设可能还有一个筛选条件，比如要筛选最近一周登录过的用户，一般大家肯定希望跟活跃用户交友了，这样能尽快收到反馈，对应后台sql可能是这样：

where  province=xx and city=xx and sex in ('female','male') and age>=xx and age<=xx and latest_login_time>= xx

那我们是否能把 latest_login_time 字段也加入索引了？比如  (province,city,sex,hobby,age,latest_login_time) ，显然是不行的，那怎么来优化这种情况了？其实我们可以试着再设计一个字段is_login_in_latest_7_days，用户如果一周内有登录值就为1，否则为0，那么我们就可以把索引设计成 (province,city,sex,hobby,is_login_in_latest_7_days,age)  来满足上面那种场景了！

一般来说，通过这么一个多字段的索引是能够过滤掉绝大部分数据的，就保留小部分数据下来基于磁盘文件进行order by语句的排序，最后基于limit进行分页，那么一般性能还是比较高的。

不过有时可能用户会这么来查询，就查下受欢迎度较高的女性，比如sql：where  sex = 'female'  order by score limit xx,xx，那么上面那个索引是很难用上的，不能把太多的字段以及太多的值都用 in 语句拼接到sql里的，那怎么办了？其实我们可以再设计一个辅助的联合索引，比如 (sex,score)，这样就能满足查询要求了。

以上就是给大家讲的一些索引设计的思路了，核心思想就是，尽量利用一两个复杂的多字段联合索引，抗下你80%以上的查询，然后用一两个辅助索引尽量抗下剩余的一些非典型查询，保证这种大数据量表的查询尽可能多的都能充分利用索引，这样就能保证你的查询速度和性能了！



# 自增主键

## 为什么用自增主键，有什么好处



如果表使用自增主键，那么每次插入新的记录，记录就会顺序添加到当前索引节点的后续位置，当一页写满，就会自动开辟一个新的页。





如果不是自增主键，那么可能会在中间插入，学过数据结构的同学都知道，在中间插入，B+树为了维持平衡，引起B+树的节点分裂（数变高，IO次数增多，效率降低）。总的来说用自增主键是可以提高查询和插入的性能。







# mysqsql查询慢



## 刷脏页

**可能正在刷内存的脏页到磁盘中。**（磁盘中的数据和内存中的数据有不一致的）



**内存页不够用了**

> 随着不断的查询，内存页中会只有脏页，这个时候如果再来一个查询，需要分配若干个内存页给它，但是在分配之前需要将之前的脏页刷盘，也就是将对应页的数据更新到磁盘上。
>
> 刷脏页就需要对磁盘进行写入了，涉及到磁盘IO，这个效率是比较慢的。
>
> 如果一个sql读取到的数据比较大，那么会需要大量的内存页来存放，那么等待的时间就更长了，就会让用户感觉出查询变慢了。
> 







**redolog日志满了**

> 因为数据更新的时候，需要写redolog。当redolog满了，会先将一部分redolog对应的内存中的数据脏页刷新到磁盘上。
>
> MySQL不会利用redolog中记录的变更信息来刷新磁盘，redolog中记录的变更信息只有在崩溃恢复的时候才会使用。
>
> MySQL会根据redolog记录中对应的数据页号找到内存中对应的内存页，然后将对应的内存页刷新到磁盘上。
>
> 因为内存中已经存在了对应的数据，如果使用redolog中的变更信息的话，还需要重新分配对应的内存，然后将信息从redolog文件读取到内存中，redolog文件是存储在磁盘上的，读取效率太低了。
>
> 如果在内存中找不到对应的数据页，就说明之前由于内存页紧张，已经把对应的数据脏页刷到磁盘上了，直接跳过就ok了。
>







## 等待锁

当更新的时候需要将对应的行锁定，如果有另一个事务在当前事务前面也更新了该行，加了锁，并且一直没有释放锁，那么当前事务就会一直等待，直到超时或其他事务释放了锁。





## 索引



### 没走索引，走了全表扫描

### 索引没设计好

### sql没设计好



慢查询被记录在慢查询日志里。可以检测慢查询日志查出执行慢的sql。

































































# mysql主从复制

MySQL 的主从复制是依赖于 binlog 的，也就是记录 MySQL 上的所有变化并以二进制形式保存在磁盘上二进制日志文件。主从复制就是将 binlog 中的数据从主库传输到从库上，一般这个过程是异步的，即主库上的操作不会等待 binlog 同步的完成。







**主从复制的过程是这样的：**首先从库在连接到主节点时会创建一个 IO 线程，用以请求主库更新的 binlog，并且把接收到的 binlog 信息写入一个叫做 relay log 的日志文件中，而主库也会创建一个 log dump 线程来发送 binlog 给从库；同时，从库还会创建一个 SQL 线程读取 relay log 中的内容，并且在从库中做回放，最终实现主从的一致性。这是一种比较常见的主从复制方式。







在这个方案中，使用独立的 log dump 线程是一种异步的方式，可以避免对主库的主体更新流程产生影响，而从库在接收到信息后并不是写入从库的存储中，是写入一个 relay log，是避免写入从库实际存储会比较耗时，最终造成从库和主库延迟变长。





![image-20220720102803238](../../images/mysql%E4%BC%98%E5%8C%96/image-20220720102803238.png)







你会发现，基于性能的考虑，主库的写入流程并没有等待主从同步完成就会返回结果，那么在极端的情况下，比如说主库上 binlog 还没有来得及刷新到磁盘上就出现了磁盘损坏或者机器掉电，就会导致 binlog 的丢失，最终造成主从数据的不一致。**不过，这种情况出现的概率很低，对于互联网的项目来说是可以容忍的。**













# 两阶段提交

在 MySQL 的 InnoDB 存储引擎中，开启 binlog 的情况下，MySQL 会同时维护 binlog 日志与 InnoDB 的 redo log，为了保证这两个日志的一致性，MySQL 使用了**内部 XA 事务**（是的，也有外部 XA 事务，跟本文不太相关，我就不介绍了），内部 XA 事务由 binlog 作为协调者，存储引擎是参与者。







# mysql分库分表

![image-20220921220231696](../../images/mysql%E4%BC%98%E5%8C%96/image-20220921220231696.png)







## 水平拆分

水平拆分是指数据表行的拆分，表的行数超过200万行时，就会变慢，这时可以把一张的表的数据拆成多张表来存放。



**水平拆分的优点：**
◆表关联基本能够在数据库端全部完成；
◆不会存在某些超大型数据量和高负载的表遇到瓶颈的问题；
◆应用程序端整体架构改动相对较少；
◆事务处理相对简单；
◆只要切分规则能够定义好，基本上较难遇到扩展性限制；

**水平切分的缺点：**
◆切分规则相对更为复杂，很难抽象出一个能够满足整个数据库的切分规则；
◆后期数据的维护难度有所增加，人为手工定位数据更困难；
◆应用系统各模块耦合度较高，可能会对后面数据的迁移拆分造成一定的困难。



##  垂直拆分

垂直拆分是指数据表列的拆分，把一张列比较多的表拆分为多张表。表的记录并不多，但是字段却很长，表占用空间很大，检索表的时候需要执行大量的IO，严重降低了性能。这时需要把大的字段拆分到另一个表，并且该表与原表是一对一的关系。







**垂直切分的优点**
◆ 数据库的拆分简单明了，拆分规则明确；
◆ 应用程序模块清晰明确，整合容易；
◆ 数据维护方便易行，容易定位；

**垂直切分的缺点**
◆ 部分表关联无法在数据库级别完成，需要在程序中完成；
◆ 对于访问极其频繁且数据量超大的表仍然存在性能平静，不一定能满足要求；
◆ 事务处理相对更为复杂；
◆ 切分达到一定程度之后，扩展性会遇到限制；
◆ 过读切分可能会带来系统过渡复杂而难以维护。



