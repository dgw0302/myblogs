---
title: sql题
categories:
  - 数据库
tags:
  - mysql
abbrlink: 56288
date: 2022-5-4 19:54:20
cover : https://img.php.cn/upload/article/000/000/024/61505c2a73cf1419.jpg 
---







# 前提



where里面不能用聚合函数（例如count()、sum()、avg()、min()、max()；）







# 1.**总共8个班级、输出每个班级男生的数学前三名              和女生的语文前三名**





```mysql
第一种写法

SELECT g1.stu_id, g1.class_id, g1.gender, g1.math_score 
FROM grade g1 
WHERE g1.gender = 'm' 
AND EXISTS (//查的是g1，和g2做比较，根据g2筛选g1,  比g1大的记录不超过3条的g1,最后找的就是前三名但是是无序的，最后再排序
   SELECT count(*) 
   FROM grade g2 
   WHERE g2.gender = 'm' 
    AND g2.math_score >= g1.math_score 
    AND g2.class_id = g1.class_id 
   GROUP BY
    g2.class_id //每个班
   HAVING count(*) <= 3//有找count小于等于3的g1 
)
ORDER BY class_id, math_score DESC//三条结果记录排序


第二种写法

select 

g1.字段

from//子查询
(
       select stu_id,
       class_id,
       gender,
       math_score,
       ROW_NUMBER k() OVER (PARTITION BY class_id ORDER BY math_score desc )  n //rank为每一行分配一个排名，字段n就是排名
    
       ROW_NUMBER() OVER(PARTITION BY 班级分组 ORDER BY 数学成绩 降序 ) AS n//按照每个分组，给每一行数据一个编号，也就是排名	
    
       ROW_NUMBER() OVER(PARTITION BY 班级分组 ORDER BY 数学成绩 DESC（降序）) AS n
       
       from grade
   		 where gender = 'm'
    
)  g1
where g1.n <= 3 ;




select
g1.name

from (
    
    select
     stu_id,
     class_id,
     gender,
    math_score,
    ROW_NUMBER() OVER (PARTITION BY calss_id ORDER BY math_score DESC) index
    FROM grede
    where gender = 'm';


) g1
where g1.index <= 3;







select 
g1.name 
from (
    select 
    dsad,
    dasd,
    ROW_NUMBER() OVER(PARTITION BY class_id ORDER BY math_score DESC ) AS n
    from
    表
    where gender = 'm'
) g1
where g1.n <= 3;









```





# 2.使用SQL查询出每门课程的成绩均大于80分的学生姓名





![img](../../images/sql%E9%A2%98/20190510154328425.png)



```java
SELECT s.name
FROM student s
GROUP BY s.name
Having MIN(s.score)>=80
```

Having使用聚合函数min,最小的分数大于80，那肯定所有的课程都大于80了。





# 3.从一个表中筛选出所有 name 字段出现两次以上的数据



## 错误示范（这波属实大意了）

```mysql
select * from 表 where name > 2;
```





## 正确示范

第一次竟然想的是上面的做法，实在可耻，可耻，不禁思考，sql学到哪里去了。

```mysql
select * from 表 GROUP BY name HAVING count(name) > 1;

```





# 4.筛选出每个poi下，score值前3的name

![image-20220810130942550](../../images/sql%E9%A2%98/image-20220810130942550.png)



```mysql
SELECT temp.*  

FROM
    (select  t1.*, ROW_NUMBER（） OVER （PARTITION  BY poi  ORDER BY score DESC）AS row_num FROM t1) temp
WHERE
    temp.row_num <= 3;
```











# **5.sql查询重复订单号查询重复的订单编号**



```mysql
SELECT
	orderNumber,
	COUNT(orderNumber) AS cou
FROM
	orderdetails
GROUP BY
	orderNumber
HAVING
	cou > 2
ORDER BY
	cou ASC;



```

















# 6.找到所有电影中共同的演员




**每个电影有不同的演员**

**找到所有电影中共同的演员**

> 【管理员】大四 金融转个屁码乙己 13:25:41
> 你group by演员
>
> 【管理员】大四 金融转个屁码乙己 13:25:56
>  count Distinct 电影
>
> 【管理员】大四 金融转个屁码乙己 13:26:12
> 等于count dsitinct所有的电影
>
> 【管理员】大四 金融转个屁码乙己 13:26:13
> 即可





```
select 演员


from 
	演员表
    		

```



![image-20220902133906055](../../images/sql%E9%A2%98/image-20220902133906055.png)











![image-20220902133919007](../../images/sql%E9%A2%98/image-20220902133919007.png)











![image-20220902133924237](../../images/sql%E9%A2%98/image-20220902133924237.png)

















# mysql8.0窗口函数



## ROW_NUMBER 函数



**row_number()从1开始，为每条分组记录返回一个数字**



MySQL 中的 ROW_NUMBER() 函数用于返回其分区内每一行的序列号。它是一种窗口函数。行号从 1 开始到分区中存在的行数。



```
ROW_NUMBER over (partition by <用于分组的列名>
                order by <用于排序的列名>)
```



## RANK()函数

MySQL `RANK()` 函数返回当前行所在的分区内的排名，从 1 开始，但有间隔。

也就是说，相同的值具有相同的排名，但是下一个不同的值的排名采用 [`row_number()`](https://www.sjkjc.com/mysql-ref/row_number/) 编号。比如，如果有 2 个第一名，那么第三位的排名是 `3`。这与 [`dense_rank()`](https://www.sjkjc.com/mysql-ref/dense_rank/) 函数是不同的 。









## 区分

1.1 区别RANK，DENSE_RANK和ROW_NUMBER
RANK并列跳跃排名，并列即相同的值，相同的值保留重复名次，遇到下一个不同值时，跳跃到总共的排名。
DENSE_RANK并列连续排序，并列即相同的值，相同的值保留重复名次，遇到下一个不同值时，依然按照连续数字排名。
ROW_NUMBER连续排名，即使相同的值，依旧按照连续数字进行排名。
区
log.csdn.net/u011726005/article/details/94592866













#  **找出每个学校GPA最低的同学**



![image-20220928135129761](../../images/sql%E9%A2%98/image-20220928135129761.png)

```mysql
SELECT
    temp.device_id,temp.university,temp.gpa

FROM (
   SELECT *, 
    ROW_NUMBER() OVER (PARTITION  BY university ORDER BY gpa ) AS n
    FROM
    user_profile

) AS temp
WHERE n = 1
ORDER BY university;
```







```




SELECT
	


FROM
l1 AS le JOIN l2 AS o ON le.next = o.next;


FROM l1 AS e LEFT JOIN x AS n ON e.x = x.l'



WHERE

GROUP BY

HAVING sum(score) > 10//having可以使用聚合h

ORDERY BY 字段 DESC降序



	





```























# 建立索引的命令（前缀索引）



```

直接建立
caeate INDEX 索引名 ON 表名



建立前缀索引
alter table user add key(字段（字段长度）)




```























