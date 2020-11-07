---
title:
  MySQL优化
tags:
  [MySQL]
categories:
	[数据库]
---

# 事务

事务指的是满足 ACID 特性的一组操作，可以通过 Commit 提交一个事务，也可以使用 Rollback 进行回滚

# MySQL优化

数据库的处理是主要的性能瓶颈

慢在：

磁盘操作为主

数据量越来越大



优化的操作

1.  设计数据库，数据表，选择字段，存储引擎
2.  利用好MySQL服务器提供的功能，索引等
3.  MySQL集群，负载均衡，读写分离
4.  SQL语句优化

## 字段设计

字段类型选择，设计规范，范式，常见设计案例

**存储IP地址**

38.71.123.233  

`varchar(15)` 常规做法

`int unsigned`  每一位都是0-255，使用 1个字节(tinyint)就可以存储，节省空间，ip运算快，但是需要额外的转换，实际上为了空间去这样做是没有意义的，空间是最不值钱的

相关MySQL函数

INET_ATON(str)，address to number

INET_NTOA(number)，number to address

**原则：尽量使用整数表示字符串**

例如MySQL内部的枚举(单选)类型和集合(多选)类型

gender enum(男，女，保密)

`insert into user(gender) values (男)` 实际存的是 1

`insert into user(gender) values (女)` 实际存的是 2

实际中enum和set很少去用，维护成本较高 ，可以用关联表替代

**金额，价格，统计数据**

对数据精度要求较高！

典型方法 price decimal(8,2) ; **定点数**，有两位小数的定点数，可以支持很大的数，不会精度丢失，浮点数会导致精度丢失

还有一个方案：price int, bigint. 整数。 **小单位，大数额**，整数的计算没有精度问题

定长类型：存储空间类型，int float double char date time datetime year timestamp 运算效率高

**原则：尽量使用小的数据类型和指定短的长度**

**原则：尽量使用not null**

null在mysql中需要额外的处理，运算也需要特殊的处理，比如`null=null`和`null<>null`返回的结果是一样的

MySQL中每条记录都需要额外的存储空间，表示每个字段是否为null。因此通常使用特殊的数据进行占位，比如

`int not null default 0`、`string not null default ''`

**原则：字段注释要完整**

gender int comment '性别'

**原则：单张表的字段数量不宜过多**

**原则：可以预留字段**

## 关联表的设计

### 一对一

如商品的基本信息（item）和商品的详细信息（item_intro），通常使用相同的主键或者增加一个额外的字段，

### 一对多，多对一



### 多对多

#### 第一范式

字段原子性，字段不可再分割。

> 关系型数据库，默认满足第一范式

注意比较容易出错的一点，在一对多的设计中使用逗号分隔多个外键，这种方法虽然存储方便，但不利于维护和索引（比如查找带标签java的文章）

| id   | subject      | tag_id           |
| ---- | ------------ | ---------------- |
| 23   | MySQL5.7发布 | 32,56,33,334,567 |

类似的如果要查找带有某个tag的文章就比较费劲，而且效率很低，最好的方式是使用关联表，将article和tag关联起来

### 第二范式

主键：可以唯一标识记录的字段或者字段集合。

| course_name | course_class | weekday（周几） | course_teacher |
| ----------- | ------------ | --------------- | -------------- |
| MySQL       | 教育大楼1525 | 周一            | 张三           |
| Java        | 教育大楼1521 | 周三            | 李四           |
| MySQL       | 教育大楼1521 | 周五            | 张三           |

依赖：A字段可以确定B字段，则B字段依赖A字段。比如知道了下一节课是数学课，就能确定任课老师是谁。于是**周几**和**下一节课**和就能构成复合主键，能够确定去哪个教室上课，任课老师是谁等。但我们常常增加一个id作为主键，而消除对主键的部分依赖。

对主键的部分依赖：某个字段依赖**复合主键**中的一部分。

解决方案：新增一个独立字段作为主键。

- 独立数据独立建表
- 表中存在与业务无关的ID主键
- 表之间的关系由关联字段（关联表）表示

优势：减少数据冗余，



## 存储引擎的对比

|                                                              | MyISAM                                          | Innodb                                   |
| ------------------------------------------------------------ | ----------------------------------------------- | ---------------------------------------- |
| 文件格式                                                     | 数据和索引是分别存储的，数据.MYD，索引.MYI      | 数据和索引是集中存储的，.ibd             |
| 文件能否移动                                                 | 能，一张表就对应.frm、MYD、MYI3个文件           | 否，因为关联的还有data下的其它文件       |
| 记录存储顺序                                                 | 按记录插入顺序保存，永远在后面插入              | 按主键大小有序保存，需要排序             |
| 空间碎片（删除记录并flush table 表名之后，表文件大小不变）   | 产生。定时整理：使用命令optimize table 表名实现 | 不产生                                   |
| 事务                                                         | 不支持                                          | 支持                                     |
| 外键                                                         | 不支持                                          | 支持                                     |
| 全文索引                                                     | 支持                                            | 5.7后支持                                |
| 锁支持（锁是避免资源争用的一个机制，MySQL锁对用户几乎是透明的） | 表级锁定                                        | 行级锁定、表级锁定，锁定粒度小并发能力高 |

## SQL执行慢

1. 查询语句写的烂
2.  索引失效
3.  关联查询太多
4.  服务器参数，连接池





手写顺序

```java
select distinct
	<select_list>
from
	<left_table> 
<join_type> join <right_table> on 
	<join_condition>
where
	<where_condition>
group by
	<group_by_list>
having
	<having_condition>
order by
	<order_by_condition>
limit <offset>,<rows>
```

机读

```java
from
	<left_table>
on
	<join_condition>
<join_type> join
	<right_table>
where
	<where_condition>
group by
	<group_by_list>
having
	<having_condition>
select
	<select_list>
order by
	<order_by_condition>
limit
	offset,rows

```

## 关联查询

![runoob](<https://www.runoob.com/wp-content/uploads/2019/01/sql-join.png>)

### 数据库表

**tbl_dept表**

| id   | deptName | locAdd |
| ---- | -------- | ------ |
| 1    | 技术部   | 11     |
| 2    | 美工部   | 12     |
| 3    | 总裁办   | 13     |
| 4    | 人力资源 | 14     |
| 5    | 后勤组   | 15     |

**tbl_emp表**

| id   | name  | deptId |
| ---- | ----- | ------ |
| 1    | jack  | 1      |
| 2    | tom   | 1      |
| 3    | alice | 2      |
| 4    | john  | 3      |
| 5    | faker | 4      |
| 6    | mlxg  | 99     |
| 7    | ning  | 100    |

### 左连接

**sql语句**

```sql
SELECT * FROM
	tbl_dept dept
LEFT JOIN tbl_emp emp ON dept.id = emp.deptId
```

**查询结果**

```java
mysql> SELECT * FROM
    ->  tbl_dept dept
    -> LEFT JOIN tbl_emp emp ON dept.id = emp.deptId;
+----+--------------+--------+------+-------+--------+
| id | deptName     | locAdd | id   | name  | deptId |
+----+--------------+--------+------+-------+--------+
|  1 | 技术部       | 11     |    1 | jack  |      1 |
|  1 | 技术部       | 11     |    2 | tom   |      1 |
|  2 | 美工部       | 12     |    3 | alice |      2 |
|  3 | 总裁办       | 13     |    4 | john  |      3 |
|  4 | 人力资源     | 14     |    5 | faker |      4 |
|  5 | 后勤组       | 15     | NULL | NULL  |   NULL |
+----+--------------+--------+------+-------+--------+
6 rows in set (0.00 sec)
```

### 右连接

**sql语句**

```sql
SELECT * FROM
	tbl_dept dept
RIGHT  JOIN tbl_emp emp ON dept.id = emp.deptId
```

**查询结果**

```java
mysql> SELECT * FROM
    ->  tbl_dept dept
    -> RIGHT  JOIN tbl_emp emp ON dept.id = emp.deptId;
+------+--------------+--------+----+-------+--------+
| id   | deptName     | locAdd | id | name  | deptId |
+------+--------------+--------+----+-------+--------+
|    1 | 技术部       | 11     |  1 | jack  |      1 |
|    1 | 技术部       | 11     |  2 | tom   |      1 |
|    2 | 美工部       | 12     |  3 | alice |      2 |
|    3 | 总裁办       | 13     |  4 | john  |      3 |
|    4 | 人力资源     | 14     |  5 | faker |      4 |
| NULL | NULL        | NULL   |  6 | mlxg  |     99 |
| NULL | NULL        | NULL   |  7 | ning  |    100 |
+------+--------------+--------+----+-------+--------+
7 rows in set (0.00 sec)
```

### 内连接

**sql语句**

```sql
SELECT * FROM
	tbl_dept dept
INNER  JOIN tbl_emp emp ON dept.id = emp.deptId
```

**查询结果**

```java
mysql> SELECT * FROM
    ->  tbl_dept dept
    -> INNER  JOIN tbl_emp emp ON dept.id = emp.deptId;
+----+--------------+--------+----+-------+--------+
| id | deptName     | locAdd | id | name  | deptId |
+----+--------------+--------+----+-------+--------+
|  1 | 技术部       | 11     |  1 | jack  |      1 |
|  1 | 技术部       | 11     |  2 | tom   |      1 |
|  2 | 美工部       | 12     |  3 | alice |      2 |
|  3 | 总裁办       | 13     |  4 | john  |      3 |
|  4 | 人力资源     | 14     |  5 | faker |      4 |
+----+--------------+--------+----+-------+--------+
5 rows in set (0.00 sec)
```

### 左表独有

**sql语句**

```sql
SELECT
	*
FROM
	tbl_dept dept
LEFT JOIN tbl_emp emp ON dept.id = emp.deptId
WHERE
	emp.deptId IS NULL
```

**查询结果**

```java
mysql> SELECT
    ->  *
    -> FROM
    ->  tbl_dept dept
    -> LEFT JOIN tbl_emp emp ON dept.id = emp.deptId
    -> WHERE
    ->  emp.deptId IS NULL;
+----+-----------+--------+------+------+--------+
| id | deptName  | locAdd | id   | name | deptId |
+----+-----------+--------+------+------+--------+
|  5 | 后勤组     | 15     | NULL | NULL |   NULL |
+----+-----------+--------+------+------+--------+
1 row in set (0.00 sec)
```

### 右表独有

**sql语句**

```sql
SELECT
	*
FROM
	tbl_dept dept
RIGHT  JOIN tbl_emp emp ON dept.id = emp.deptId
WHERE
	dept.id IS NULL
```

**查询结果**

```java
mysql> SELECT
    ->  *
    -> FROM
    ->  tbl_dept dept
    -> RIGHT  JOIN tbl_emp emp ON dept.id = emp.deptId
    -> WHERE
    ->  dept.id IS NULL;
+------+----------+--------+----+------+--------+
| id   | deptName | locAdd | id | name | deptId |
+------+----------+--------+----+------+--------+
| NULL | NULL     | NULL   |  6 | mlxg |     99 |
| NULL | NULL     | NULL   |  7 | ning |    100 |
+------+----------+--------+----+------+--------+
2 rows in set (0.00 sec)
```

### 左右表独有

**sql语句**

```sql
SELECT
	*
FROM
	tbl_dept dept
RIGHT  JOIN tbl_emp emp ON dept.id = emp.deptId
WHERE
	dept.id IS NULL
UNION
SELECT
	*
FROM
	tbl_dept dept
LEFT JOIN tbl_emp emp ON dept.id = emp.deptId
WHERE
	emp.deptId IS NULL
```

**查询结果**

```java
mysql> SELECT
    ->  *
    -> FROM
    ->  tbl_dept dept
    -> RIGHT  JOIN tbl_emp emp ON dept.id = emp.deptId
    -> WHERE
    ->  dept.id IS NULL
    -> UNION
    -> SELECT
    ->  *
    -> FROM
    ->  tbl_dept dept
    -> LEFT JOIN tbl_emp emp ON dept.id = emp.deptId
    -> WHERE
    ->  emp.deptId IS NULL;
+------+-----------+--------+------+------+--------+
| id   | deptName  | locAdd | id   | name | deptId |
+------+-----------+--------+------+------+--------+
| NULL | NULL      | NULL   |    6 | mlxg |     99 |
| NULL | NULL      | NULL   |    7 | ning |    100 |
|    5 | 后勤组     | 15     | NULL | NULL |   NULL |
+------+-----------+--------+------+------+--------+
3 rows in set (0.00 sec)
```

### 全外连接

很显然mysql是**不支持全外连接**的，但是我们可以通过其他的途径来实现需求

**sql语句**

```sql
SELECT
	*
FROM
	tbl_dept dept
RIGHT JOIN tbl_emp emp ON dept.id = emp.deptId
WHERE
	dept.id IS NULL
UNION
	SELECT
		*
	FROM
		tbl_dept dept
	LEFT JOIN tbl_emp emp ON dept.id = emp.deptId
	WHERE
		emp.deptId IS NULL
	UNION
		SELECT
			*
		FROM
			tbl_dept dept
		INNER JOIN tbl_emp emp ON dept.id = emp.deptId
```

很明显，上面是我一开始xjb写的，好久没写过都忘了，union其实带有去重的功能

```sql
SELECT
	*
FROM
	tbl_dept dept
RIGHT JOIN tbl_emp emp ON dept.id = emp.deptId
UNION
	SELECT
		*
	FROM
		tbl_dept dept
	LEFT JOIN tbl_emp emp ON dept.id = emp.deptId
```

**查询结果**

```java
mysql> SELECT
    ->  *
    -> FROM
    ->  tbl_dept dept
    -> RIGHT JOIN tbl_emp emp ON dept.id = emp.deptId
    -> UNION
    ->  SELECT
    ->          *
    ->  FROM
    ->          tbl_dept dept
    ->  LEFT JOIN tbl_emp emp ON dept.id = emp.deptId;
+------+--------------+--------+------+-------+--------+
| id   | deptName     | locAdd | id   | name  | deptId |
+------+--------------+--------+------+-------+--------+
|    1 | 技术部       | 11     |    1 | jack  |      1 |
|    1 | 技术部       | 11     |    2 | tom   |      1 |
|    2 | 美工部       | 12     |    3 | alice |      2 |
|    3 | 总裁办       | 13     |    4 | john  |      3 |
|    4 | 人力资源     | 14     |    5 | faker |      4 |
| NULL | NULL        | NULL   |    6 | mlxg  |     99 |
| NULL | NULL        | NULL   |    7 | ning  |    100 |
|    5 | 后勤组       | 15     | NULL | NULL  |   NULL |
+------+--------------+--------+------+-------+--------+
8 rows in set (0.00 sec)
```

## 索引

索引是一种数据结构，在插入一条记录时，它从记录中提取（建立了索引的字段的）字段值作为该数据结构的元素，该数据结构中的元素被有序组织，因此在建立了索引的字段上搜索记录时能够借助二分查找提高搜索效率；此外每个元素还有一个指向它所属记录（数据库表记录一般保存在磁盘上）的指针，因此索引与数据库表的关系可类比于字典中目录与正文的关系，且目录的篇幅（索引所占的存储空间存储空间）很小，简单来说索引就是一种 `排好序的快速查找数据结构` 

### 优势

提高了数据检索的效率，降低排序的成本

### 劣势

- 索引也是一张表，该表保存了主键和索引字段，并指向实体表的记录，所以索引列也是要占用空间的
- 索引大大提高了查询速度，同时却会降低更新表的速度，如对表进行INSERT，UPDATE和DELETE，因为更新表的时候，MySQL不仅要保存数据，还要保存一下索引文件每次更新添加了索引列的字段，都会调整因为更新带来的键值变化后的索引信息
- 索引只是提高效率的一个因素，如果你的MySQL有大数据量的表，就需要花时间建立优秀的索引

**常用索引**

- 主键索引（`primary key`），只能作用于一个字段（列），字段值不能为`null`且不能重复。
- 唯一索引（`unique key`），只能作用于一个字段，字段值可以为`null`但不能重复
- 普通索引（`key`），可以作用于一个或多个字段，对字段值没有限制。为一个字段建立索引时称为单值索引，为多个字段同时建立索引时称为复合索引（提取多个字段值组合而成）

那些情况建立索引

1. 主键自动建立唯一索引
2.  频繁作为查询条件的字段应该建立索引
3.  查询中与其他表关联的字段，外键关系建立索引
4.  频繁更新的字段不适合创建SQL
5.  Where条件里用不到的字段不建立索引
6.  单键/组合索引的选择问题(高并发下倾向于创建组合索引)
7.  查询中排序的字段，排序字段若通过索引去访问将大大提高排序速度
8.  查询中统计或分组字段

  不适合建立索引

1. 表记录太少
2. 经常增删改的表
3. 数据重复，且分布平均的表字段，比如人的性别 

## SQL执行计划——Explain

通过`EXPLAIN`分析某条SQL语句执行时的如下特征：

- 表的读取顺序（涉及到多张表时）
- 数据读取操作的操作类型
- 哪些索引可以使用
- 哪些索引被实际使用
- 表之间的引用
- 每张表有多少行被优化器查询

![mark](http://static.imlgw.top/blog/20190928/xvkgzEPqn9XF.png?imageslim)

### id

select查询的序列号，包含一组数字，表示查询中执行select子句或操作表的顺序。根据`id`是否相同可以分为下列三种情况

- **id相同，执行顺序从上至下**

  ![mark](https://uploadfiles.nowcoder.com/files/20190603/8222772_1559492690822_006zweohly1g3mvcbjdekj30p804y75f.jpg)

  上表三个表项按照从上至下的循序执行，则上图中读表循序为t1-->t3-->t2

- **id不同**

  比如嵌套查询，id的序号会由内而外递增，id的值越大，优先级越高，越先被执行

  ![sql](https://uploadfiles.nowcoder.com/files/20190603/8222772_1559492691097_006zweohly1g3mvjaq4ugj30j906cmx6.jpg)

   读表顺序：t3-->t1-->t2

- **id有相同有不同**

  ![mark](http://static.imlgw.top/blog/20190929/d1r8eBXVm1zF.png?imageslim)

  相同的按照相同的规则来，不同的按照不同的规则来，这里`deriverd2` 说明这张表是id为2的表的衍生表，也就是t3的衍生表

### select_type

- **SIMPLE** 最简单的`select`查询，不包含子查询或者UNION
- **PRIMARY** 查询中包含任何复杂的子查询，最外层查询则被标记为PRIMARY
- **SUBQUERY** 在`select`或`where`列表中包含的子查询
- **DERIVED** 在`from`子句中的子查询被标记为`DERIVED`（衍生），MySQL会递归执行这些子查询, 把结果放在临时表里
- **UNION** 若第二个`select` 出现在`UNION` 后，则被标记为`UNION`，简单来说就是右侧的`select`
- **UNION RESULT**  从UNION表中获取的结果

### table

表示查询涉及的表或衍生表

### partitions

该列显示的为分区表命中的分区情况。非分区表该字段为空（null）

### type

type表示的是访问类型，是一个很重要的性能指标，常见的结果值从好到坏依次是

> system > const > eq_ref > ref > fulltext > ref_or_null > index_merge > unique_subquery > index_subquery > range > index > ALL 

一般常见的就只有下面几种

 `system>const>eq_ref>ref>range>index>ALL`

**system**

表只有一行记录（等于系统表），这是const类型的特例，基本不可能出现

**const**

表示通过索引一次就找到了，const用于比较`primary key` ，或者`unique key` 因为只匹配一行数据，所以很快如将主键置于where列表中，MySQL就能将查询转换为一个常量

**eq_ref**

唯一性索引扫描，对于每个索引键，表中只有一条记录与之匹配，常见于主键或唯一索引扫描

**ref**

非唯一性索引扫描，返回匹配某个单独值的所有行，本质上也是一种索引访问，它返回所有匹配某个单独值的行，然而，他可能会找到多个符合条件的行，所以他应该属于查找和扫描的混合体

**range**

根据索引的有序性检索特定范围内的行，通常出现在`between、<、>、in`等范围检索中

**index**

全索引扫描，在索引中扫描，只需读取索引数据，index和all都是读全表，但是index是从索引表中读取，而all是从全表上读取的

**all**

全表扫描，相比于`index` 会多很多磁盘的IO，性能会比较低

> 一般情况下需要保证查询到达range级别，最好能达到ref

### possible_keys 

指出MySQL能使用哪个索引在表中找到记录，查询涉及到的字段上若存在索引，则该索引将被列出，**但不一定被查询使用**

### key

实际使用的索引，如果为NULL，没创建索引或者索引失效，查询中如果出现了覆盖索引，则该索引仅出现在key列表中，而不出现在possible_keys 中

### key_len

表示索引中每个元素最大字节数，可通过该列计算查询中使用的索引的长度，key_len显示的值为索引字段的最大可能长度，并非实际使用长度，即key_len是**根据表定义计算而得**，不是通过表内检索出的

```java
CREATE TABLE `member` (
 `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
 `name` varchar(20) DEFAULT NULL,
 `age` tinyint(3) unsigned DEFAULT NULL,
 PRIMARY KEY (`id`),
 KEY `name` (`name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

INSERT INTO `member` (`id`, `name`, `age`) VALUES (NULL, 'fdipzone', '18'), (NULL, 'jim', '19'), (NULL, 'tom', '19');
```

查看**explain**

```java
mysql> explain select * from `member` where name='fdipzone';
+----+-------------+--------+------------+------+---------------+------+---------+-------+-----
| id | select_type | table  | partitions | type | possible_keys | key  | key_len | ref   | rows | filtered | Extra |
+----+-------------+--------+------------+------+---------------+------+---------+-------+-----
|  1 | SIMPLE      | member | NULL       | ref  | name          | name | 63      | const |    1 
+----+-------------+--------+------------+------+---------------+------+---------+-------+-----
........
```

使用utf8字符，一个字符占用3个字节，那么`key_len`应该是 20*3=60，但是结果多个3个字节，首先使用变长字段需要额外增加2个字节，使用NULL需要额外增加1个字节，因此对于是索引的字段，最好使用定长和NOT NULL定义，提高性能

### ref

显示哪一列或常量被拿来与索引列进行比较以从表中检索行，比如上面的const

```java
mysql> explain select * from `member` where name='fdipzone';
+----+-------------+--------+------------+------+---------------+------+---------+-------+-----
| id | select_type | table  | partitions | type | possible_keys | key  | key_len | ref   | rows | filtered | Extra |
+----+-------------+--------+------------+------+---------------+------+---------+-------+-----
|  1 | SIMPLE      | member | NULL       | ref  | name          | name | 63      | const |    1 
+----+-------------+--------+------------+------+---------------+------+---------+-------+-----
........
```

### row

根据表统计信息及索引选用情况，大致估算出找到所需记录所需要读取的行数，所以这一列肯定是越小越好

### Extra

包含不适合在其他列中显示但十分重要的额外信息

- **Using filesort** 见名知意，使用文件排序，说明mysql会对数据使用一个外部的索引排序，而不是按照表内的索引顺序进行读取，可想而知这会很影响效率

- **Using temporary** 使用了临时表保存中间结果，MySQL在对查询结果聚合时使用临时表。常见于排序 `order by`和分组查询 `group by`

- **Using index**  表示相应的select操作中使用了覆盖索引(Covering Index)，避免直接访问表的数据行

  > **覆盖索引：**其实翻译为索引覆盖会更加直观，具体体现就是select列中的数据只用从索引中就可以获得，不用通过行再去数据表上获取，也就是**需要查询的列被所建的索引覆盖了**
  >
  > 如果使用覆盖索引，一定要注意select列表中只取出需要的列，不要使用`select *` 因为如果将所有字段一起做索引会导致文件过大，查询性能下降

- **Using where** 查询使用了`where`语句

- **Using join buffer** 使用了连接缓存

- **Impossible where** 看名字就知道了where子句的值总是`false` 类似于 `select * from table where name="lgw" and name="zzz"` 显然这是个无效的where语句

## 索引优化

## 索引失效

