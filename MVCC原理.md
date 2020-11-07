## MVCC原理

### 版本链

我们前边说过，对于使用`InnoDB`存储引擎的表来说，它的聚簇索引记录中都包含两个必要的隐藏列（`row_id`并不是必要的，我们创建的表中有主键或者非NULL的UNIQUE键时都不会包含`row_id`列）：

- `trx_id`：每次一个事务对某条聚簇索引记录进行改动时，都会把该事务的`事务id`赋值给`trx_id`隐藏列。
- `roll_pointer`：每次对某条聚簇索引记录进行改动时，都会把旧的版本写入到`undo日志`中，然后这个隐藏列就相当于一个指针，可以通过它来找到该记录修改前的信息。

比方说我们的表`hero`现在只包含一条记录：

```
mysql> SELECT * FROM hero;
+--------+--------+---------+
| number | name   | country |
+--------+--------+---------+
|      1 | 刘备   | 蜀      |
+--------+--------+---------+
1 row in set (0.07 sec)
```

假设插入该记录的`事务id`为`80`，那么此刻该条记录的示意图如下所示：



![image_1d8oab1ubb7v5f41j2pai21co19.png-22.4kB](https://user-gold-cdn.xitu.io/2019/4/19/16a337f526c95a9e?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



> 小贴士： 实际上insert undo只在事务回滚时起作用，当事务提交后，该类型的undo日志就没用了，它占用的Undo Log Segment也会被系统回收（也就是该undo日志占用的Undo页面链表要么被重用，要么被释放）。虽然真正的insert undo日志占用的存储空间被释放了，但是roll_pointer的值并不会被清除，roll_pointer属性占用7个字节，第一个比特位就标记着它指向的undo日志的类型，如果该比特位的值为1时，就代表着它指向的undo日志类型为insert undo。所以我们之后在画图时都会把insert undo给去掉，大家留意一下就好了。

假设之后两个`事务id`分别为`100`、`200`的事务对这条记录进行`UPDATE`操作，操作流程如下：



![image_1d8obbc861ulkpt3no31gecrho16.png-92.3kB](https://user-gold-cdn.xitu.io/2019/4/19/16a337f52a913b66?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



> 小贴士： 能不能在两个事务中交叉更新同一条记录呢？哈哈，这不就是一个事务修改了另一个未提交事务修改过的数据，沦为了脏写了么？InnoDB使用锁来保证不会有脏写情况的发生，也就是在第一个事务更新了某条记录后，就会给这条记录加锁，另一个事务再次更新时就需要等待第一个事务提交了，把锁释放之后才可以继续更新。关于锁的更多细节我们后续的文章中再唠叨哈～

每次对记录进行改动，都会记录一条`undo日志`，每条`undo日志`也都有一个`roll_pointer`属性（`INSERT`操作对应的`undo日志`没有该属性，因为该记录并没有更早的版本），可以将这些`undo日志`都连起来，串成一个链表，所以现在的情况就像下图一样：



![image_1d8po6kgkejilj2g4t3t81evm20.png-81.7kB](https://user-gold-cdn.xitu.io/2019/4/19/16a33e277a98dbec?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



对该记录每次更新后，都会将旧值放到一条`undo日志`中，就算是该记录的一个旧版本，随着更新次数的增多，所有的版本都会被`roll_pointer`属性连接成一个链表，我们把这个链表称之为`版本链`，版本链的头节点就是当前记录最新的值。另外，每个版本中还包含生成该版本时对应的`事务id`，这个信息很重要，我们稍后就会用到。

### ReadView

对于使用`READ UNCOMMITTED`隔离级别的事务来说，由于可以读到未提交事务修改过的记录，所以直接读取记录的最新版本就好了；对于使用`SERIALIZABLE`隔离级别的事务来说，设计`InnoDB`的大叔规定使用加锁的方式来访问记录（加锁是啥我们后续文章中说哈）；对于使用`READ COMMITTED`和`REPEATABLE READ`隔离级别的事务来说，都必须保证读到已经提交了的事务修改过的记录，也就是说假如另一个事务已经修改了记录但是尚未提交，是不能直接读取最新版本的记录的，核心问题就是：需要判断一下版本链中的哪个版本是当前事务可见的。为此，设计`InnoDB`的大叔提出了一个`ReadView`的概念，这个`ReadView`中主要包含4个比较重要的内容：

- `m_ids`：表示在生成`ReadView`时当前系统中活跃的读写事务的`事务id`列表。

- `min_trx_id`：表示在生成`ReadView`时当前系统中活跃的读写事务中最小的`事务id`，也就是`m_ids`中的最小值。

- `max_trx_id`：表示生成`ReadView`时系统中应该分配给下一个事务的`id`值。

  > 小贴士： 注意max_trx_id并不是m_ids中的最大值，事务id是递增分配的。比方说现在有id为1，2，3这三个事务，之后id为3的事务提交了。那么一个新的读事务在生成ReadView时，m_ids就包括1和2，min_trx_id的值就是1，max_trx_id的值就是4。

- `creator_trx_id`：表示生成该`ReadView`的事务的`事务id`。

  > 小贴士： 我们前边说过，只有在对表中的记录做改动时（执行INSERT、DELETE、UPDATE这些语句时）才会为事务分配事务id，否则在一个只读事务中的事务id值都默认为0。

有了这个`ReadView`，这样在访问某条记录时，只需要按照下边的步骤判断记录的某个版本是否可见：

- 如果被访问版本的`trx_id`属性值与`ReadView`中的`creator_trx_id`值相同，意味着当前事务在访问它自己修改过的记录，所以该版本可以被当前事务访问。
- 如果被访问版本的`trx_id`属性值小于`ReadView`中的`min_trx_id`值，表明生成该版本的事务在当前事务生成`ReadView`前已经提交，所以该版本可以被当前事务访问。
- 如果被访问版本的`trx_id`属性值大于或等于`ReadView`中的`max_trx_id`值，表明生成该版本的事务在当前事务生成`ReadView`后才开启，所以该版本不可以被当前事务访问。
- 如果被访问版本的`trx_id`属性值在`ReadView`的`min_trx_id`和`max_trx_id`之间，那就需要判断一下`trx_id`属性值是不是在`m_ids`列表中，如果在，说明创建`ReadView`时生成该版本的事务还是活跃的，该版本不可以被访问；如果不在，说明创建`ReadView`时生成该版本的事务已经被提交，该版本可以被访问。

如果某个版本的数据对当前事务不可见的话，那就顺着版本链找到下一个版本的数据，继续按照上边的步骤判断可见性，依此类推，直到版本链中的最后一个版本。如果最后一个版本也不可见的话，那么就意味着该条记录对该事务完全不可见，查询结果就不包含该记录。

在`MySQL`中，`READ COMMITTED`和`REPEATABLE READ`隔离级别的的一个非常大的区别就是它们生成ReadView的时机不同。我们还是以表`hero`为例来，假设现在表`hero`中只有一条由`事务id`为`80`的事务插入的一条记录：

```
mysql> SELECT * FROM hero;
+--------+--------+---------+
| number | name   | country |
+--------+--------+---------+
|      1 | 刘备   | 蜀      |
+--------+--------+---------+
1 row in set (0.07 sec)
```

接下来看一下`READ COMMITTED`和`REPEATABLE READ`所谓的生成ReadView的时机不同到底不同在哪里。

#### READ COMMITTED —— 每次读取数据前都生成一个ReadView

比方说现在系统里有两个`事务id`分别为`100`、`200`的事务在执行：

```
# Transaction 100
BEGIN;

UPDATE hero SET name = '关羽' WHERE number = 1;

UPDATE hero SET name = '张飞' WHERE number = 1;
# Transaction 200
BEGIN;

# 更新了一些别的表的记录
...
```

> 小贴士： 再次强调一遍，事务执行过程中，只有在第一次真正修改记录时（比如使用INSERT、DELETE、UPDATE语句），才会被分配一个单独的事务id，这个事务id是递增的。所以我们才在Transaction 200中更新一些别的表的记录，目的是让它分配事务id。

此刻，表`hero`中`number`为`1`的记录得到的版本链表如下所示：



![image_1d8poeb056ck1d552it4t91aro2d.png-63.7kB](https://user-gold-cdn.xitu.io/2019/4/19/16a33e277e11d7b8?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



假设现在有一个使用`READ COMMITTED`隔离级别的事务开始执行：

```
# 使用READ COMMITTED隔离级别的事务
BEGIN;

# SELECT1：Transaction 100、200未提交
SELECT * FROM hero WHERE number = 1; # 得到的列name的值为'刘备'
```

这个`SELECT1`的执行过程如下：

- 在执行`SELECT`语句时会先生成一个`ReadView`，`ReadView`的`m_ids`列表的内容就是`[100, 200]`，`min_trx_id`为`100`，`max_trx_id`为`201`，`creator_trx_id`为`0`。
- 然后从版本链中挑选可见的记录，从图中可以看出，最新版本的列`name`的内容是`'张飞'`，该版本的`trx_id`值为`100`，在`m_ids`列表内，所以不符合可见性要求，根据`roll_pointer`跳到下一个版本。
- 下一个版本的列`name`的内容是`'关羽'`，该版本的`trx_id`值也为`100`，也在`m_ids`列表内，所以也不符合要求，继续跳到下一个版本。
- 下一个版本的列`name`的内容是`'刘备'`，该版本的`trx_id`值为`80`，小于`ReadView`中的`min_trx_id`值`100`，所以这个版本是符合要求的，最后返回给用户的版本就是这条列`name`为`'刘备'`的记录。

之后，我们把`事务id`为`100`的事务提交一下，就像这样：

```
# Transaction 100
BEGIN;

UPDATE hero SET name = '关羽' WHERE number = 1;

UPDATE hero SET name = '张飞' WHERE number = 1;

COMMIT;
```

然后再到`事务id`为`200`的事务中更新一下表`hero`中`number`为`1`的记录：

```
# Transaction 200
BEGIN;

# 更新了一些别的表的记录
...

UPDATE hero SET name = '赵云' WHERE number = 1;

UPDATE hero SET name = '诸葛亮' WHERE number = 1;
```

此刻，表`hero`中`number`为`1`的记录的版本链就长这样：



![image_1d8poudrjdrk4k0i22bj10g82q.png-78.6kB](https://user-gold-cdn.xitu.io/2019/4/19/16a33e277f08dc3c?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



然后再到刚才使用`READ COMMITTED`隔离级别的事务中继续查找这个`number`为`1`的记录，如下：

```
# 使用READ COMMITTED隔离级别的事务
BEGIN;

# SELECT1：Transaction 100、200均未提交
SELECT * FROM hero WHERE number = 1; # 得到的列name的值为'刘备'

# SELECT2：Transaction 100提交，Transaction 200未提交
SELECT * FROM hero WHERE number = 1; # 得到的列name的值为'张飞'
```

这个`SELECT2`的执行过程如下：

- 在执行`SELECT`语句时会又会单独生成一个`ReadView`，该`ReadView`的`m_ids`列表的内容就是`[200]`（`事务id`为`100`的那个事务已经提交了，所以再次生成快照时就没有它了），`min_trx_id`为`200`，`max_trx_id`为`201`，`creator_trx_id`为`0`。
- 然后从版本链中挑选可见的记录，从图中可以看出，最新版本的列`name`的内容是`'诸葛亮'`，该版本的`trx_id`值为`200`，在`m_ids`列表内，所以不符合可见性要求，根据`roll_pointer`跳到下一个版本。
- 下一个版本的列`name`的内容是`'赵云'`，该版本的`trx_id`值为`200`，也在`m_ids`列表内，所以也不符合要求，继续跳到下一个版本。
- 下一个版本的列`name`的内容是`'张飞'`，该版本的`trx_id`值为`100`，小于`ReadView`中的`min_trx_id`值`200`，所以这个版本是符合要求的，最后返回给用户的版本就是这条列`name`为`'张飞'`的记录。

以此类推，如果之后`事务id`为`200`的记录也提交了，再此在使用`READ COMMITTED`隔离级别的事务中查询表`hero`中`number`值为`1`的记录时，得到的结果就是`'诸葛亮'`了，具体流程我们就不分析了。总结一下就是：使用READ COMMITTED隔离级别的事务在每次查询开始时都会生成一个独立的ReadView。

#### REPEATABLE READ —— 在第一次读取数据时生成一个ReadView

对于使用`REPEATABLE READ`隔离级别的事务来说，只会在第一次执行查询语句时生成一个`ReadView`，之后的查询就不会重复生成了。我们还是用例子看一下是什么效果。

比方说现在系统里有两个`事务id`分别为`100`、`200`的事务在执行：

```
# Transaction 100
BEGIN;

UPDATE hero SET name = '关羽' WHERE number = 1;

UPDATE hero SET name = '张飞' WHERE number = 1;
# Transaction 200
BEGIN;

# 更新了一些别的表的记录
...
```

此刻，表`hero`中`number`为`1`的记录得到的版本链表如下所示：



![image_1d8pt2nd6moqtjn12hibgj91f37.png-60.9kB](https://user-gold-cdn.xitu.io/2019/4/19/16a33e277f5bfb75?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



假设现在有一个使用`REPEATABLE READ`隔离级别的事务开始执行：

```
# 使用REPEATABLE READ隔离级别的事务
BEGIN;

# SELECT1：Transaction 100、200未提交
SELECT * FROM hero WHERE number = 1; # 得到的列name的值为'刘备'
```

这个`SELECT1`的执行过程如下：

- 在执行`SELECT`语句时会先生成一个`ReadView`，`ReadView`的`m_ids`列表的内容就是`[100, 200]`，`min_trx_id`为`100`，`max_trx_id`为`201`，`creator_trx_id`为`0`。
- 然后从版本链中挑选可见的记录，从图中可以看出，最新版本的列`name`的内容是`'张飞'`，该版本的`trx_id`值为`100`，在`m_ids`列表内，所以不符合可见性要求，根据`roll_pointer`跳到下一个版本。
- 下一个版本的列`name`的内容是`'关羽'`，该版本的`trx_id`值也为`100`，也在`m_ids`列表内，所以也不符合要求，继续跳到下一个版本。
- 下一个版本的列`name`的内容是`'刘备'`，该版本的`trx_id`值为`80`，小于`ReadView`中的`min_trx_id`值`100`，所以这个版本是符合要求的，最后返回给用户的版本就是这条列`name`为`'刘备'`的记录。

之后，我们把`事务id`为`100`的事务提交一下，就像这样：

```
# Transaction 100
BEGIN;

UPDATE hero SET name = '关羽' WHERE number = 1;

UPDATE hero SET name = '张飞' WHERE number = 1;

COMMIT;
```

然后再到`事务id`为`200`的事务中更新一下表`hero`中`number`为`1`的记录：

```
# Transaction 200
BEGIN;

# 更新了一些别的表的记录
...

UPDATE hero SET name = '赵云' WHERE number = 1;

UPDATE hero SET name = '诸葛亮' WHERE number = 1;
```

此刻，表`hero`中`number`为`1`的记录的版本链就长这样：



![image_1d8ptbc339kdk0b1du3nef6s03k.png-78.2kB](https://user-gold-cdn.xitu.io/2019/4/19/16a33e277f809c36?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



然后再到刚才使用`REPEATABLE READ`隔离级别的事务中继续查找这个`number`为`1`的记录，如下：

```
# 使用REPEATABLE READ隔离级别的事务
BEGIN;

# SELECT1：Transaction 100、200均未提交
SELECT * FROM hero WHERE number = 1; # 得到的列name的值为'刘备'

# SELECT2：Transaction 100提交，Transaction 200未提交
SELECT * FROM hero WHERE number = 1; # 得到的列name的值仍为'刘备'
```

这个`SELECT2`的执行过程如下：

- 因为当前事务的隔离级别为`REPEATABLE READ`，而之前在执行`SELECT1`时已经生成过`ReadView`了，所以此时直接复用之前的`ReadView`，之前的`ReadView`的`m_ids`列表的内容就是`[100, 200]`，`min_trx_id`为`100`，`max_trx_id`为`201`，`creator_trx_id`为`0`。
- 然后从版本链中挑选可见的记录，从图中可以看出，最新版本的列`name`的内容是`'诸葛亮'`，该版本的`trx_id`值为`200`，在`m_ids`列表内，所以不符合可见性要求，根据`roll_pointer`跳到下一个版本。
- 下一个版本的列`name`的内容是`'赵云'`，该版本的`trx_id`值为`200`，也在`m_ids`列表内，所以也不符合要求，继续跳到下一个版本。
- 下一个版本的列`name`的内容是`'张飞'`，该版本的`trx_id`值为`100`，而`m_ids`列表中是包含值为`100`的`事务id`的，所以该版本也不符合要求，同理下一个列`name`的内容是`'关羽'`的版本也不符合要求。继续跳到下一个版本。
- 下一个版本的列`name`的内容是`'刘备'`，该版本的`trx_id`值为`80`，小于`ReadView`中的`min_trx_id`值`100`，所以这个版本是符合要求的，最后返回给用户的版本就是这条列`c`为`'刘备'`的记录。

也就是说两次`SELECT`查询得到的结果是重复的，记录的列`c`值都是`'刘备'`，这就是`可重复读`的含义。如果我们之后再把`事务id`为`200`的记录提交了，然后再到刚才使用`REPEATABLE READ`隔离级别的事务中继续查找这个`number`为`1`的记录，得到的结果还是`'刘备'`，具体执行过程大家可以自己分析一下。

### MVCC小结

从上边的描述中我们可以看出来，所谓的`MVCC`（Multi-Version Concurrency Control ，多版本并发控制）指的就是在使用`READ COMMITTD`、`REPEATABLE READ`这两种隔离级别的事务在执行普通的`SELECT`操作时访问记录的版本链的过程，这样子可以使不同事务的`读-写`、`写-读`操作并发执行，从而提升系统性能。`READ COMMITTD`、`REPEATABLE READ`这两个隔离级别的一个很大不同就是：生成ReadView的时机不同，READ COMMITTD在每一次进行普通SELECT操作前都会生成一个ReadView，而REPEATABLE READ只在第一次进行普通SELECT操作前生成一个ReadView，之后的查询操作都重复使用这个ReadView就好了。

> 小贴士： 我们之前说执行DELETE语句或者更新主键的UPDATE语句并不会立即把对应的记录完全从页面中删除，而是执行一个所谓的delete mark操作，相当于只是对记录打上了一个删除标志位，这主要就是为MVCC服务的，大家可以对比上边举的例子自己试想一下怎么使用。 另外，所谓的MVCC只是在我们进行普通的SEELCT查询时才生效，截止到目前我们所见的所有SELECT语句都算是普通的查询，至于啥是个不普通的查询，我们稍后再说哈～

## 为什么RR不能完全解决幻读问题

```mysql
# 事务T1，REPEATABLE READ隔离级别下
mysql> BEGIN;
Query OK, 0 rows affected (0.00 sec)

mysql> SELECT * FROM hero WHERE number = 30;
Empty set (0.01 sec)

# 此时事务T2执行了：INSERT INTO hero VALUES(30, 'g关羽', '魏'); 并提交

#T1更新了number=30的记录，trxid变为T1的事务id
mysql> UPDATE hero SET country = '蜀' WHERE number = 30; 

Query OK, 1 row affected (0.01 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> SELECT * FROM hero WHERE number = 30;
+--------+---------+---------+
| number | name    | country |
+--------+---------+---------+
|     30 | g关羽   | 蜀      |
+--------+---------+---------+
1 row in set (0.01 sec)
```

RR隔离级别下，T1先查询了number=30的记录，随后T2插入了一题number为30的记录并提交，随后T1对number=32的这一条记录做了update，这就导致了这条记录的隐藏属性`trxid` 变成了T1的事务id，我们回头看一下ReadView读取的规则

> - 如果被访问版本的`trx_id`属性值与`ReadView`中的`creator_trx_id`值相同，意味着当前事务在访问它自己修改过的记录，所以该版本可以被当前事务访问。
> - .....

显然，T1可以读取该记录，发生了幻读！

## 幻读和不可重复读的区别

不可重复读的重点是修改，幻读的重点在于新增或者删除。

## 加锁分析

```java
CREATE TABLE hero (
    number INT,
    name VARCHAR(100),
    country varchar(100),
    PRIMARY KEY (number),
    KEY idx_name (name)
) Engine=InnoDB CHARSET=utf8;

INSERT INTO hero VALUES
    (1, 'l刘备', '蜀'),
    (3, 'z诸葛亮', '蜀'),
    (8, 'c曹操', '魏'),
    (15, 'x荀彧', '魏'),
    (20, 's孙权', '吴');
```

我们这里把语句分为3种大类：普通的`SELECT`语句、锁定读的语句、`INSERT`语句，我们分别看一下。

### **普通的SELECT语句**

普通的`SELECT`语句在：

- `READ UNCOMMITTED`隔离级别下，不加锁，直接读取记录的最新版本，可能发生`脏读`、`不可重复读`和`幻读`问题。
- `READ COMMITTED`隔离级别下，不加锁，在每次执行普通的`SELECT`语句时都会生成一个`ReadView`，这样解决了`脏读`问题，但没有解决`不可重复读`和`幻读`问题。
- `REPEATABLE READ`隔离级别下，不加锁，只在第一次执行普通的`SELECT`语句时生成一个`ReadView`，这样把`脏读`、`不可重复读`和`幻读`问题都解决了。

- `SERIALIZABLE`隔离级别下，需要分为两种情况讨论：

- - 在系统变量`autocommit=0`时，也就是禁用自动提交时，普通的`SELECT`语句会被转为`SELECT ... LOCK IN SHARE MODE`这样的语句，也就是在读取记录前需要先获得记录的`S锁`，具体的加锁情况和`REPEATABLE READ`隔离级别下一样，我们后边再分析。

  - 在系统变量`autocommit=1`时，也就是启用自动提交时，普通的`SELECT`语句并不加锁，只是利用`MVCC`来生成一个`ReadView`去读取记录。

    为啥不加锁呢？因为启用自动提交意味着一个事务中只包含一条语句，一条语句也就没有啥`不可重复读`、`幻读`这样的问题了。

### **锁定读的语句**

我们把下边四种语句放到一起讨论：

- 语句一：`SELECT ... LOCK IN SHARE MODE;`
- 语句二：`SELECT ... FOR UPDATE;`
- 语句三：`UPDATE ...`
- 语句四：`DELETE ...`

我们说`语句一`和`语句二`是`MySQL`中规定的两种`锁定读`的语法格式，而`语句三`和`语句四`由于在执行过程需要首先定位到被改动的记录并给记录加锁，也可以被认为是一种`锁定读`。

#### **READ UNCOMMITTED/READ COMMITTED隔离级别下**

在`READ UNCOMMITTED`下语句的加锁方式和`READ COMMITTED`隔离级别下语句的加锁方式基本一致，所以就放到一块儿说了。值得注意的是，采用`加锁`方式解决并发事务带来的问题时，其实`脏读`和`不可重复读`在任何一个隔离级别下都不会发生（因为`读-写`操作需要排队进行）。

##### **对于使用主键进行等值查询的情况**

使用`SELECT ... LOCK IN SHARE MODE`来为记录加锁，比方说：

```mysql
SELECT * FROM hero WHERE number = 8 LOCK IN SHARE MODE;
```

这个语句执行时只需要访问一下聚簇索引中`number`值为`8`的记录，所以只需要给它加一个`S型正经记录锁`就好了，如图所示：

![img](https://mmbiz.qpic.cn/mmbiz/RLmbWWew55GM3XXtto9wTRRBb4WCDIUkFjXSynwywNXUsdFuTzMazMMM2T4zI7jSUHllJXC9JYAiax48yFwJiaLg/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

使用`SELECT ... FOR UPDATE`来为记录加锁，比方说：

```mysql
SELECT * FROM hero WHERE number = 8 FOR UPDATE;
```

这个语句执行时只需要访问一下聚簇索引中`number`值为`8`的记录，所以只需要给它加一个`X型正经记录锁`就好了，如图所示：

![img](https://mmbiz.qpic.cn/mmbiz/RLmbWWew55GM3XXtto9wTRRBb4WCDIUkwsVic92nrQia80BgKdxD5K7Brv2icSR2h3McLYKy5ZSZBaEDu0hP2XcNQ/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- 使用`UPDATE ...`来为记录加锁，比方说：

  ```mysql
  UPDATE hero SET country = '汉' WHERE number = 8;
  ```

  这条`UPDATE`语句并没有更新二级索引列，加锁方式和上边所说的`SELECT ... FOR UPDATE`语句一致。

  如果`UPDATE`语句中更新了二级索引列，比方说：

  ```mysql
  UPDATE hero SET name = 'cao曹操' WHERE number = 8;
  ```

  该语句的实际执行步骤是首先更新对应的`number`值为`8`的聚簇索引记录，再更新对应的二级索引记录，所以加锁的步骤就是：

- 1. 为`number`值为`8`的聚簇索引记录加上`X型正经记录锁`（该记录对应的）。
  2. 为该聚簇索引记录对应的`idx_name`二级索引记录（也就是`name`值为`'c曹操'`，`number`值为`8`的那条二级索引记录）加上`X型正经记录锁`。

- 画个图就是这样：

- 

- ![img](https://mmbiz.qpic.cn/mmbiz/RLmbWWew55GM3XXtto9wTRRBb4WCDIUkSucbP6v7dANly9HwHK4kUVXqlbANPHhNJ2ibSBct3OJmbRVF9MusaQQ/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- 使用`DELETE ...`来为记录加锁，比方说：

  ```mysql
  DELETE FROM hero WHERE number = 8;
  ```

  我们平时所说的“DELETE表中的一条记录”其实意味着对聚簇索引和所有的二级索引中对应的记录做`DELETE`操作，本例子中就是要先把`number`值为`8`的聚簇索引记录执行`DELETE`操作，然后把对应的`idx_name`二级索引记录删除，所以加锁的步骤和上边更新带有二级索引列的`UPDATE`语句一致，就不画图了。

##### **对于使用主键进行范围查询的情况**

- 使用`SELECT ... LOCK IN SHARE MODE`来为记录加锁，比方说：

  ```
  SELECT * FROM hero WHERE number <= 8 LOCK IN SHARE MODE;
  ```

  这个语句看起来十分简单，但它的执行过程还是有一丢丢小复杂的：

- 1. 先到聚簇索引中定位到满足`number <= 8`的第一条记录，也就是`number`值为`1`的记录，然后为其加锁。

  2. 判断一下该记录是否符合`索引条件下推`中的条件。

     我们前边介绍过一个称之为`索引条件下推`（ `Index Condition Pushdown`，简称`ICP`）的功能，也就是把查询中与被使用索引有关的查询条件下推到存储引擎中判断，而不是返回到`server`层再判断。不过需要注意的是，`索引条件下推`只是为了减少回表次数，也就是减少读取完整的聚簇索引记录的次数，从而减少`IO`操作。而对于`聚簇索引`而言不需要回表，它本身就包含着全部的列，也起不到减少`IO`操作的作用，所以设计`InnoDB`的大叔们规定这个`索引条件下推`特性只适用于`二级索引`。也就是说在本例中与被使用索引有关的条件是：`number <= 8`，而`number`列又是聚簇索引列，所以本例中并没有符合`索引条件下推`的查询条件，自然也就不需要判断该记录是否符合`索引条件下推`中的条件。

  3. 判断一下该记录是否符合范围查询的边界条件

     因为在本例中是利用主键`number`进行范围查询，设计`InnoDB`的大叔规定每从聚簇索引中取出一条记录时都要判断一下该记录是否符合范围查询的边界条件，也就是`number <= 8`这个条件。如果符合的话将其返回给`server层`继续处理，否则的话需要释放掉在该记录上加的锁，并给`server层`返回一个查询完毕的信息。

     对于`number`值为`1`的记录是符合这个条件的，所以会将其返回到`server层`继续处理。

  4. 将该记录返回到`server层`继续判断。

     `server层`如果收到存储引擎层提供的查询完毕的信息，就结束查询，否则继续判断那些没有进行`索引条件下推`的条件，在本例中就是继续判断`number <= 8`这个条件是否成立。噫，不是在第3步中已经判断过了么，怎么在这又判断一回？是的，设计`InnoDB`的大叔采用的策略就是这么简单粗暴，把凡是没有经过`索引条件下推`的条件都需要放到`server`层再判断一遍。如果该记录符合剩余的条件（没有进行`索引条件下推`的条件），那么就把它发送给客户端，不然的话需要释放掉在该记录上加的锁。

  5. 然后刚刚查询得到的这条记录（也就是`number`值为`1`的记录）组成的单向链表继续向后查找，得到了`number`值为`3`的记录，然后重复第`2`，`3`，`4`、`5`这几个步骤。

但是这个过程有个问题，就是当找到`number`值为`8`的那条记录的时候，还得向后找一条记录（也就是`number`值为`15`的记录），在存储引擎读取这条记录的时候，也就是上述的第`1`步中，就得为这条记录加锁，然后在第3步时，判断该记录不符合`number <= 8`这个条件，又要释放掉这条记录的锁，这个过程导致`number`值为`15`的记录先被加锁，然后把锁释放掉，过程就是这样：

- ![img](https://mmbiz.qpic.cn/mmbiz/RLmbWWew55GM3XXtto9wTRRBb4WCDIUkZMEzurcc6BVL2EfqCVK9my7PNeGbOoDpG3zlDicsaJMCIxFm4IcQyWg/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- 

- 这个过程有意思的一点就是，如果你先在事务`T1`中执行：

- ```mysql
  # 事务T1
  BEGIN;
  SELECT * FROM hero WHERE number <= 8 LOCK IN SHARE MODE;
  ```

- 然后再到事务`T2`中执行：

- ```mysql
  # 事务T2
  BEGIN;
  SELECT * FROM hero WHERE number = 15 FOR UPDATE;
  ```

- 是没有问题的，因为在`T2`执行时，事务`T1`已经释放掉了`number`值为`15`的记录的锁，但是如果你先执行`T2`，再执行`T1`，由于`T2`已经持有了`number`值为`15`的记录的锁，事务`T1`将因为获取不到这个锁而等待。

- - 使用`SELECT ... FOR UPDATE`语句来为记录加锁：

    和`SELECT ... FOR UPDATE`语句类似，只不过加的是`X型正经记录锁`。

  - 使用`UPDATE ...`来为记录加锁，比方说：

    ```
    UPDATE hero SET country = '汉' WHERE number >= 8;
    ```

    这条`UPDATE`语句并没有更新二级索引列，加锁方式和上边所说的`SELECT ... FOR UPDATE`语句一致。

    如果`UPDATE`语句中更新了二级索引列，比方说：

    ```
    UPDATE hero SET name = 'cao曹操' WHERE number >= 8;
    ```

    这时候会首先更新聚簇索引记录，再更新对应的二级索引记录，所以加锁的步骤就是：

  - 1. 为`number`值为`8`的聚簇索引记录加上`X型正经记录锁`。
    2. 然后为上一步中的记录索引记录对应的`idx_name`二级索引记录加上`X型正经记录锁`。
    3. 为`number`值为`15`的聚簇索引记录加上`X型正经记录锁`。
    4. 然后为上一步中的记录索引记录对应的`idx_name`二级索引记录加上`X型正经记录锁`。
    5. 为`number`值为`20`的聚簇索引记录加上`X型正经记录锁`。
    6. 然后为上一步中的记录索引记录对应的`idx_name`二级索引记录加上`X型正经记录锁`。

  - 画个图就是这样：

  - ![img](https://mmbiz.qpic.cn/mmbiz/RLmbWWew55GM3XXtto9wTRRBb4WCDIUkicwyzWY2dhgXtwa6s4Hib04lh0O3JMTUarQpkicwJHSAjzfxYRLoHaa8Q/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

  - 如果是下边这个语句：

  - ```
    UPDATE hero SET namey = '汉' WHERE number <= 8;
    ```

  - 则会对`number`值为`1`、`3`、`8`聚簇索引记录以及它们对应的二级索引记录加`X型正经记录锁`，加锁顺序和上边语句中的加锁顺序类似，都是先对一条聚簇索引记录加锁后，再给对应的二级索引记录加锁。之后会继续对`number`值为`15`的聚簇索引记录加锁，但是随后`InnoDB`存储引擎判断它不符合边界条件，随即会释放掉该聚簇索引记录上的锁（注意这个过程中没有对`number`值为`15`的聚簇索引记录对应的二级索引记录加锁）。具体示意图就不画了。

  - 