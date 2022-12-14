---
title: mysql sql语句执行原理
categories:
  - 数据库
tags:
  - mysql
abbrlink: 56288
date: 2022-03-6 19:54:20
cover : https://img.php.cn/upload/article/000/000/024/61505c2a73cf1419.jpg
---





# mysql逻辑架构图

![image-20220303132652117](../../images/mysql%20sql%E8%AF%AD%E5%8F%A5%E6%89%A7%E8%A1%8C%E5%8E%9F%E7%90%86/image-20220303132652117.png)



大体来说，MySQL 可以分为 **Server** 层和**存储引擎层**两部分。



- **Server**

  server层包括连接器、查询缓存、分析器、优化器、执行器等（mysql8.0之后取消了查询缓存）

- **存储引擎层**

  存储引擎层负责数据的存储和提取





# 连接器

第一步，你会先连接到这个数据库上，这时候接待你的就是连接器。连接器负责跟客户端建立连接、获取权限、维持和管理连接,连接器来进行身份验证和权限认证。

连接命令一般是这么写的：

```mysql
mysql -h$ip -P$port -u$user -p
```

mysql默认的是长连接，默认是8个小时，即使这个连接用户没有操作，也不会断开连接。



数据库里面，长连接是指连接成功后，如果客户端持续有请求，则一直使用同一个连接。短连接则是指每次执行完很少的几次查询就断开连接，下次查询再重新建立一个。



**注意：**

一个用户成功建立连接后，即使你用管理员账号对这个用户的权限做了修改，也不会影响已经存在连接的权 限。修改完成后，只有再新建的连接才会使用新的权限设置。用户的权限表在系统表空间的mysql的user表中。



# 查询缓存



连接建立完成后，你就可以执行 select 语句了。执行逻辑就会来到第二步：查询缓存。

MySQL 拿到一个查询请求后，会先到查询缓存看看，之前是不是执行过这条语句。之前执行过的语句及其结果可能会以 key-value 对的形式，被直接缓存在内存中。key 是查询的语句，value 是查询的结果。如果你的查询能够直接在这个缓存中找到 key，那么这个 value 就会被直接返回给客户端。

如果语句不在查询缓存中，就会继续后面的执行阶段。执行完成后，执行结果会被存入查询缓存中。如果查询命中缓存，MySQL 不需要执行后面的复杂操作，就可以直接返回结果，这个效率会很高。

- **不建议使用缓存，很鸡肋**

只要有对一个表的更新，这个表上所有的查询缓存都会被清空，所以8.0以后mysql取消了查询缓存。



# 分析器

如果没有命中查询缓存，就要开始真正执行语句了。首先，MySQL 需要知道你要做什么，因此需要对 SQL 语句做解析。

- **怎样分析的**

分析器对sql语句进行语法、词法分析来看看是否满足mysql语法。

词法语法分析看这条语句是否符合它的语法规则，并解析关键词等



# 优化器

经过了分析器，MySQL 就知道你要做什么了。在开始执行之前，还要先经过优化器的处理。



- **优化器是怎样优化的**

MySQL 会通过优化器生成该 SQL语句的执行计划，我们可以通过 explain 命令查询。然后若你用到了索引**可能**还会选择索引，它会选择最优的方案去执行，因为即使你使用了索引，在某些情况可能也没有全表扫描来的快。”



# 执行器

MySQL 通过分析器知道了你要做什么，通过优化器知道了该怎么做，于是就进入了执行器阶段，开始执行语句。



- **怎样执行的**

开始执行的时候，要先判断一下对这个表有没有执行查询的权限，如果没有，就会返回没有权限的错误。如果有权限，就打开表继续执行。打开表的时候，执行器就会根据表的引擎定义，去调用这个引擎提供的接口。





# 存储引擎

常见的存储引擎有**InnoDB**(默认) **MyISAM**  **MEMORY**

| 功能         | MyISAM | MEMORY | InnoDB |
| ------------ | ------ | ------ | ------ |
| 存储限制     | 256TB  | 内存   | 64TB   |
| 支持事务     | 否     | 否     | 是     |
| 支持全文索引 | 是     | 否     | 否     |
| 支持B+树索引 | 是     | 是     | 是     |
| 支持Hash索引 | 否     | 是     | 否     |
| 支持集群索引 | 否     | 否     | 是     |
| 支持数据索引 | 否     | 是     | 是     |
| 支持数据压缩 | 是     | 否     | 否     |
| 空间使用率   | 低     | /      | 高     |
| 支持外键     | 否     | 否     | 是     |

# 日志系统

前面是查询流程，而与查询流程不一样的是，更新流程还涉及两个重要的日志模块。

**redo log**（重做日志）和 **binlog**（归档日志）



## redo log

**redo log是存储引擎层的日志**

MySQL 里经常说到的 WAL 技术，WAL 的全称是 Write-Ahead Logging，它的关键点就是先写日志，再写磁盘



当有一条记录需要更新的时候，InnoDB 引擎就会先把记录写到 redo log（粉板）里面，并更新内存，这个时候更新就算完成了。同时，InnoDB 引擎会在适当的时候，将这个操作记录更新到磁盘里面，而这个更新往往是在系统比较空闲的时候做。



InnoDB 的 redo log 是固定大小的，比如可以配置为一组 4 个文件，每个文件的大小是 1GB，那么这块“粉板”总共就可以记录 4GB 的操作。从头开始写，写到末尾就又回到开头循环写

![image-20220303173559646](../../images/mysql%20sql%E8%AF%AD%E5%8F%A5%E6%89%A7%E8%A1%8C%E5%8E%9F%E7%90%86/image-20220303173559646.png)



write pos 是当前记录的位置，一边写一边后移，写到第 3 号文件末尾后就回到 0 号文件开头。checkpoint 是当前要擦除的位置，也是往后推移并且循环的，擦除记录前要把记录更新到数据文件。



有了 redo log，InnoDB 就可以保证即使数据库发生异常重启，之前提交的记录都不会丢失，这个能力称为**crash-safe**。



## bin log

**Server 层也有自己的日志，称为 binlog**



## 对比

1. redo log 是 InnoDB 引擎特有的；binlog 是 MySQL 的 Server 层实现的，所有引擎都可以使用。
2. redo log 是物理日志，记录的是“在某个数据页上做了什么修改”；binlog 是逻辑日志，记录的是这个语句的原始逻辑，比如“给 ID=2 这一行的 c 字段加 1 ”。
3. redo log 是循环写的，空间固定会用完；binlog 是可以追加写入的。“追加写”是指 binlog 文件写到一定大小后会切换到下一个，并不会覆盖以前的日志。



# 查询sql的执行（重点）

上面的sql的执行流程，那如果完整的论述是怎样的呢

```mysql
select name from xl_tb where 主键ID=1
```

**分析这条sql的执行**

1. 首先要我们的客户端去连接服务器，需要连接器进行身份验证和权限验证
2. 然后在mysql8.0之前，如果开启了缓存，执行这条sql的话会首先以这条sql为key去缓存查询是否有对应的value,如果有直接返回结果，mysql8.0在综合考虑其优劣之后移除了缓存
3. 如果没有缓存命中，那么接下来会利用分析器通过词法、语法分析看着条语句是否符合它的语法规则，并解决关键词。
4. 然后是到了优化器，优化器会通过其内部操作生成改sql语句的执行计划，会选择最优的执行计划。
5. 接下来到了执行器这块，执行器打开表调用存储引擎接口来进行数据的存取。





# 修改sql的执行

简单的 update 语句时的内部流程



1. 执行器先找引擎取 ID=2 这一行。ID 是主键，引擎直接用树搜索找到这一行。如果 ID=2 这一行所在的数据页本来就在内存中，就直接返回给执行器；否则，需要先从磁盘读入内存，然后再返回。
2. 执行器拿到引擎给的行数据，把这个值加上 1，比如原来是 N，现在就是 N+1，得到新的一行数据，再调用引擎接口写入这行新数据。
3. 引擎将这行新数据更新到内存中，同时将这个更新操作记录到 redo log 里面，此时 redo log 处于 prepare 状态。然后告知执行器执行完成了，随时可以提交事务。
4. 执行器生成这个操作的 binlog，并把 binlog 写入磁盘。
5. 执行器调用引擎的提交事务接口，引擎把刚刚写入的 redo log 改成提交（commit）状态，更新完成。



![image-20220303175757378](../../images/mysql%20sql%E8%AF%AD%E5%8F%A5%E6%89%A7%E8%A1%8C%E5%8E%9F%E7%90%86/image-20220303175757378.png)









# ChangeBuffer（写缓存）



WAL：先写日志再写磁盘；写日志前得先在内存更新了。

我们平常更新语句的时候，并不是每次都直接更新到磁盘里面，而是会先在内存更新数据，如果内存没有会从磁盘读数据到内存然后在内存更新数据，更新完的数据叫做脏页，会定期刷新到磁盘，这个刷新到磁盘也是会分两部，（第一步先更新到操作系统内核缓存，然后才会调用系统函数刷新回硬盘），至于什么是会刷新，有很多种情况（比如说redolog写满了，定期到了等等）。





## 介绍

上面我们遇到一种情况，当更新数据时，内存没有相应的数据，这时候就要将数据从磁盘读进内存，这个涉及随机IO访问，成本很高，而利用写缓存（Change buffer）可以减少IO操作。



changeBuffer是BufferPool里面的一个东西，主要是记录更新语句对数据页的修改信息，这样即使内存中没有要修改的信息，我们也不用将数据从磁盘读进内存了，这个changeBuffer也是会持久化到磁盘的。



**那可能还有个问题，它只是记录了对数据页的修改，但磁盘的数据还是没有真正改变啊，我们再次读修改后的数据的时候从磁盘读的是没有改变的数据啊？**

当我门再次访问被修改的数据页时，会先将数据加载进内存然后执行changebuffer里面关于这个数据页的修改也就是merge



**什么时候触发这个merge？**

除了上述数据读进内存的时候会merge,我们后台肯定也会有线程定期merge.





## **适用场景**

Change Buffer 并不是适用于所有场景，以下两种情况不适合开启 Change Buffer ：

- **1、数据库都是唯一索引**

如果数据库都是唯一索引，那么在每次操作的时候都需要判断索引是否有冲突，势必要将数据加载到缓存中对比，因此也用不到 Change Buffer。



- **2、写入一个数据后，会立刻读取它**

写入一个数据后，会立刻读取它，那么即使满足了条件，将更新先记录在 change buffer，但之后由于马上要访问这个数据页，会立即触发 merge 过程。这样随机访问 IO 的次数不会减少，反而增加了 change buffer 的维护代价。所以，对于这种业务模式来说，change buffer 反而起到了副作用。



**以下几种情况开启 Change Buffer，会使得 MySQL 数据库明显提升**：

- **1、数据库大部分是非唯一索引**
- **2、业务是写多读少**
- **3、写入数据之后并不会立即读取它**
- 

总体来说 InnoDB 的写缓存（Change Buffer）应用得当，会极大提高 MySQL 数据库的性能，使用不恰当的话，可能会适得其反。



# 两阶段提交

为什么必须有“两阶段提交”呢？这是为了让两份日志之间的逻辑一致。

将redo log的写入拆成了两个步骤：prepare和commit，这就是"两阶段提交"。





举例：执行器拿到引擎给的行数据，把这个值加上1，比如原来是N，现在就是N+1，得到新的一行数据，再调用引擎接口写入这行新数据。

​			

引擎将这行新数据更新到内存中，同时将这个更新操作记录到redo log里面，此时redo log处于**prepare**状态，然后告知执行器执行完成了，执行器然后写binlog，

提交事务之后，redo log日志由刚才的prepare状态变为**commit**状态。







































