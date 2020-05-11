- [MySQL](#mysql)
  - [了解SQL](#%e4%ba%86%e8%a7%a3sql)
    - [1.数据库基础](#1%e6%95%b0%e6%8d%ae%e5%ba%93%e5%9f%ba%e7%a1%80)
    - [2.SQL](#2sql)
  - [SQL语句](#sql%e8%af%ad%e5%8f%a5)
    - [1.检索数据(select)](#1%e6%a3%80%e7%b4%a2%e6%95%b0%e6%8d%aeselect)
    - [2.排序数据](#2%e6%8e%92%e5%ba%8f%e6%95%b0%e6%8d%ae)
    - [3.过滤数据](#3%e8%bf%87%e6%bb%a4%e6%95%b0%e6%8d%ae)
    - [4.SQL通配符](#4sql%e9%80%9a%e9%85%8d%e7%ac%a6)
    - [5.正则表达式搜索](#5%e6%ad%a3%e5%88%99%e8%a1%a8%e8%be%be%e5%bc%8f%e6%90%9c%e7%b4%a2)
    - [6.拼接](#6%e6%8b%bc%e6%8e%a5)
    - [7.聚集函数](#7%e8%81%9a%e9%9b%86%e5%87%bd%e6%95%b0)
    - [8.分组数据](#8%e5%88%86%e7%bb%84%e6%95%b0%e6%8d%ae)
    - [9.联结表](#9%e8%81%94%e7%bb%93%e8%a1%a8)
    - [10.插入数据](#10%e6%8f%92%e5%85%a5%e6%95%b0%e6%8d%ae)
    - [11.更新和删除数据](#11%e6%9b%b4%e6%96%b0%e5%92%8c%e5%88%a0%e9%99%a4%e6%95%b0%e6%8d%ae)
# MySQL

## 了解SQL

### 1.数据库基础

* 数据库  
  保存有组织的数据的容器，人们通过数据库管理软件(DBMS)来操纵数据库
* 表(table)  
  某种特定类型数据的结构化清单。例如对于顾客和订单，我们应该创建两个table储存，而不是存在同一个talbe中
  
  数据库中每个table有一个名字，称为表名，用来标示自己
* 列  
  列是表中的一个字段，所有表都是由一个或多个列组成的，每列都有对应的数据类型

  以顾客表为例，每一列储存了顾客的某一项信息
* 行  
  数据表记录是按行存储的，每个记录储存在自己的行内
* 主键(primary key)
  表内每一行都有一个唯一标识自己的ID，称为～。是一列，其值能够区分每一行

### 2.SQL

SQL是结构化查询语言(Structed Query Language)的缩写，是用来专门与数据库通信的语言

* 选择数据库  
  `use （database—name);`，必须先选择数据库，才能读取其中的数据
* 显示所有可用数据库
  `show databases;`
* 显示可用tables
  `show tables;`
* show还可以用来显示表列
  `show columuns from (customers);`
  也可以用`describe （customers)`代替

## SQL语句

### 1.检索数据(select)

select是最经常使用的SQL语句了，他可以从一个或多个表中检索信息。为了使用select，我们至少要给出想选择什么，以及从什么地方选择

* 检索单个列  
  `select (prod_name) from (products);`从products表中检索名字为prod_name的列
* 检索多个列  
  `select col1, col2, col3 from table`选择多个列输出，最后一个不加逗号
* 检索所有列  
  `select * from table`利用×通配符
* 利用distinct可以去重`select distinct col from table`，如果是多列的结果，那么去重操作也是对于多列相等才会去重
* 限制输出范围
  `select col1 from table limit (begin,) lines`，通过limit语句选择起始行(可选，默认从0开始)，以及显示的行数。

### 2.排序数据

利用`order by`子句可以将输出结果进行排序
* `order by col1, col2`按照col1和col2排序结果
* 在任意col后加desc，可以让该列降序排列
* 配合limit语句，可以输出前几的名单

### 3.过滤数据

利用where语句限定输出的数据，如`select data from table where data = 2.50`可以限定输出内容为2.50的信息

where语句支持的操作符有：= <> != < <= > >= between(在比较时，用``表示字符串匹配，否则为数值比较)

where语句还支持使用and和or进行多个条件的合并

where配合in语句可以限定多个条件，如`select data from table where data in (1, 2)`可以挑选data为1,2的字段

not用来否定之后的所有条件

### 4.SQL通配符

利用like操作符，可以指示mysql，后面的搜索模式是通配符匹配

mysql支持如下的通配符：
* %: 表示任意字符出现任意次数
* _: 表示任意字符出现一次

### 5.正则表达式搜索

在where语句中，调用regexp表示后面式子采用正则表达式，正则表达式比较复杂，这里值列举几个

* .: 表示任意字符出现一次
* |: 匹配两个字符
* []: 匹配[]框起来的某一个字符
* \\: 对于特殊字符，如.，在前面加\\转义

### 6.拼接

有时候需要得到从多个列拼接起来的字段，mysql采用Concat函数进行字符串的拼接(拼接是，可以采用L/R/Trim()函数删除两侧多余的空格)

计算出的新列是没有名字的，调用as可以给他取个别名

此外，select中，还支持不同列间的运算，包括加减乘除

### 7.聚集函数

我们有时候选取数据之后，只关心他的一些统计信息，并不关心每行的细节，有一些聚合函数可以帮助我们做到

* avg(): 某列的平均值
* count(): 某列的行数
* max(): 某列的最大值
* min(): 某列最小值
* sum(): 某列累加值

### 8.分组数据

聚集函数可以很方便的帮助我们进行统计操作，但是如果想对每组(如每个客户的统计)，就需要进行分组了。使用分组，可以让我们对每个组进行聚集计算

分组使用group by col，可以按条件进行分组

group by col还可以通过添加having来限制，从而筛选出一些想要的分组

### 9.联结表

首先，为了伸缩性好，以及避免重复，通常会信息拆分成多个表来进行储存

外键: 为了表示两个表之间的关系，在某个表中，有一列是另一个表的主键

调用select可以在两个表之间查找数据，但是当可能出现歧义时，要用完全限定名

* 内部联结: 基于两个表之间相等的映射，有两种方式：
  * 利用where t1.col = t2.col
  * 利用inner join语句，并用on给出二者的关系，` from t1 inner join t2 on contidion`
* 外部联结: 包含一些没有关联的行`t1 left/right outer join t2 on condition` left/right表示没有关联的内容从join左还是右选择

### 10.插入数据

insert也是常用的语句，有一下几种常用的方式：
* 插入完整的行  
  `insert into t1 values(1, 2, 3)`，这种方法严重依赖表的顺序，很不安全，**不推荐！！！**
* 插入部分
  正确的做法是，指定每个插入值对应的列，`insert into t1(col1, col2) values(1, 2)`
* 插入检索数据
  insert into t1 select ...

### 11.更新和删除数据

* 更新数据  
    更新数据由update语句来实现
    * 更新一个列(通过where，设置条件)`update t1 set col1 = 'sdf' where cust_id = 'id'`
    * 更新多个列，`update t1 set col1 = 'sdf'， col2 = 'dsf' where cust_id = 'id'`
    * 为了删除某个列，可以通过把它置为NULL实现
* 删除数据
    删除数据通过delete语句实现
    * 删除一行，`delete from t1 where col1 = 'val'`，删除的是一整行，如果想删除某一列，请使用update


