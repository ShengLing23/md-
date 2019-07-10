# InnoDB存储引擎-读书笔记

## 存储引擎

### InnoDB

设计目标：面向在线事务处理（OLTP)应用，支持事务

特点：采用MVCC(多版本控制)来获取高并发

​	   行锁设计，支持外键

### MyISAM

主要面向OLAP数据库应用

不支持事务

表锁设计，支持全文索引

 ## 体系架构

![1550503896375](.\img\%5CUsers%5Csurface%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5C1550503896375.png)

​	后台线程主要作用时负责刷新内存池中的数据，保证缓冲池中的内存缓存的时最新的数据。此外将以修改的数据文件刷新到磁盘文件。

### 后台线程

​	InnoDB是多线程的模型，因此后台有多个不同的后台线程，负责处理不同的任务。

#### 1、Master Thread

​	Master Thread是一个非常核心的后台线程，主要负责将缓冲池中的数据异步刷新到磁盘，保证数据的一致性，包括赃页的刷新，合并插入缓冲，undo页的回收等。

#### 2、IO Thread

​	在InnoDB存储引擎中大量使用了AIO来处理写IO请求。而IO Thread的工作主要负责这些IO请求的回调处理

* 在InnoDB 1.0版本之前共有4个IO Thread，分别是write、read、insert buffer 和 log IO Thread

```shell 
# 在Linux平台下，IO Thread的数量不能进行调整
# 但是在Windows平台下，可以通过如下参数，来增大IO Thread.
innodb_file_io_threads
```

* 在InnoDB 1.0.x版本开始，read thread和write thread 分别增大到4个，并且不再使用innodb_file_io_threads参数

  ```shell
  #分别使用参数设置
  innodb_read_io_threads
  innodb_write_io_threads
  ```

  ```mysql
  /*可以使用如下命令来观察InnoDB中的IO Thread*/
  SHOW ENGINE INNODB STATUS\G;
  ```

#### 3、Purge Thread

​	事务被提交后，其所使用的undolog可能不再需要，因此需要Purge Thread来回收已经使用并分配的undo页。

​	在InnoDB 1.1版本之前，purge操作仅在Master Thread中完成。

​	从InnoDB 1.1版本开始，purge操作可以独立到单点的线程中进行。

```shell
# 用户可以在mysql数据库的配置文件中添加如下命令来启动独立的purge thread:
# 在InnoDB 1.1版本中，purge_thread只能为1
# 从InnoDB 1.2版本开始，InnoDB支持多个Purge Thread.
[mysqld]
innodb_purge_threads=1
# 以下命令查看
show variables like 'innodb_purge_threads'\G
```


#### 4、Page  Cleaner Thread

​	Page cleaner Thread在InnoDB 1.2.X版本中引入。作用是将之前版本中的脏页的刷新操作都放入到单独的线程中来完成。

### 内存

#### 1、缓冲池

​	InnoDB存储引擎是基于磁盘存储的，并将其中的记录按照页的方式进行管理。因此可以将其视为【基于磁盘的数据库系统】。由于CPU速度与磁盘速度之间的鸿沟，基于磁盘的数据库系统通常使用【缓冲池技术】来提高数据库的整体性能。

 * 在数据库中进行读取页的操作，首先将从磁盘读取到的页放在缓冲池中，这个过程称为将页“FIX"在缓冲池中。
* 对于数据库中页的修改操作，则首先修改缓冲池中的页，然后再以一定的频率刷新到磁盘上。
* 页从缓冲池刷新回磁盘的操作，并不是在每次也发生更新时触发的。而是通过一种称之为Checkpoint的机制刷新回磁盘

```shell
#参数设置缓冲池大小
innodb_buffer_pool_size
```

![1550542409194](.\img\1550542409194.png)

​	从InnoDB 1.0.x版本开始，允许多个缓冲池实例。每个页根据哈希值平均分配到不同的缓冲池实例中。好处：减少数据库内部的资源竞争，增加数据库的并发处理能力

```shell
#配置数据库缓冲池实例
innodb_buffer_pool_instances=1
```

#### 2、LRU List、Free List和Flush List

​	数据库中的缓冲池通过【LRU】（Latest Recent Used,最近最少使用）算法来进行管理。即【最频繁使用的页在LRU列表的前端，而最少使用的页在LRU列表的尾端。当缓冲池不能存放新读取到的页时，将首先释放LRU列表中尾端的页】

​	InnoDB对传统的【LRU】算法进行了一些优化。

​	【在InnoDB的存储引擎中，LRU列表还加入了midpoint位置。新读取到页，虽然是最新访问的页，但并不直接放入LRU列表的首部，而是放入到LRU列表的midpoint位置】

```shell
# midpoint位置可以由参数
innodb_old_blocks_pct=37
# 进行控制
```

​	midpoint之后的列表称之为old列表，之前的列表称之为new列表。

-----------------------------------------------------------------------------------------------------------------------------

​	【LRU】列表用来管理已经读取的页，但是当数据库刚启动时，【LRU】列表是空的。这时页都放在【Free】列表中。

```shell
# 当需要从缓冲池分页时，首先从Free列表中查找是否有可用的空闲页，若有则将该页从Free列表删除，放入到LRU列表中。
# 否则，根据LRU算法，淘汰LRU列表末尾的页，将该内存空间分配给新的页
```

#### 3、重做日志缓冲

​	InnoDB存储引擎的内存区域除了有缓冲池外，还有重做日志缓冲(redo log buffer)。InnoDB存储引擎首先将重做日志信息先放入到这个缓冲区，然后按一定频率将其刷新到重做日志文件中。

​	重做日志缓冲一般不需要设置的很大，一般情况下，每一秒会将重做日志缓冲刷新到日志文件。

```shell
# 控制重做日志缓冲大学
innodb_log_buffer_size
```

重做日志在如下三种情况下回刷新到外部磁盘的文件中：

1、master Thread 每一秒将重做日志缓刷新

2、每个事物提交时会将重做日志缓冲刷新

3、当重做日志缓冲池剩余空间小于1/2时，重做日志会被刷新

#### 4、Checkpoint技术

## 关键特性

### 1、插入缓存（Insert Buffer）

### 2、两次写（Double Write）

### 3、自适应哈希索引（adaptive Hash Index）

### 4、异步IO

### 5、刷新邻接页

## 锁

### 什么是锁

​	锁是数据库系统区别于文件系统的一个关键性特性。

​	InnoDB存储引擎会在行级别上对表数据上锁。也会在数据库其他多个地方使用锁

### lock于latch

​	latch一般被称之为闩锁（轻量级的锁）。因为其要求的锁定时间必须非常的短。若持续的时间长，则应用的性能会变得非常的差。

​	lock的对象是事务。用来锁定的是数据库中的对象。如：表，页，行。并且一般的lock对象仅在事务commit或rollback后进行释放。

### InnoDB存储引擎中的锁

#### 锁的类型

​	InnoDB存储引擎实现了如下两种标准的锁：

* 共享锁(S Lock):允许事务读取一行数据
* 排它锁（X Lock)：允许事务删除或更新一行数据。

|      | X      | S      |
| ---- | ------ | ------ |
| X    | 不兼容 | 不兼容 |
| S    | 不兼容 | 兼容   |

​	S和X锁都是行锁，兼容是指对同一记录（row)锁的兼容

​	InnoDB存储引擎支持意向锁设计比较简练。其意向锁即为表锁。设计目的主要是为了在一个事务中揭示下一行将被请求的锁类型。支持两种意向锁：

* 意向共享锁（IS Lock）:事务想要获取一张表中的某几行的共享锁
* 意向排它锁（IX Lock)   :事务想要获取一张表中的某几行的排它锁

|      | IS     | IX     | S      | X      |
| ---- | ------ | ------ | ------ | ------ |
| IS   | 兼容   | 兼容   | 兼容   | 不兼容 |
| IX   | 兼容   | 兼容   | 不兼容 | 不兼容 |
| S    | 兼容   | 不兼容 | 兼容   | 不兼容 |
| X    | 不兼容 | 不兼容 | 不兼容 | 不兼容 |

#### 一致性非锁定读

​	一致性非锁定读（consistent nonloking read)是指InnoDB存储引擎通过多版本控制的方式来读取当前执行时间数据库中行的数据。

​	如果当前读取的行正在执行DELETE或UPDATE操作，这时读取操作不会因此区等待行上锁的释放。相反的，InnoDB存储引擎回去读取行的一个快照数据。

​	![一致性非锁定读](img\一致性非锁定读.png)

​	快照数据是指该行之前版本的数据，该实现是通过undo段来完成的。而undo段用来在事务中回滚数据。

​	非锁定读机制极大的提高了数据库的并发性。在InnoDB引擎的默认设置下，这是默认的读取方式，即读取不会占用和等待表上的锁。但是在不同的事务隔离级别下，读取的方式不同。及时都是使用非锁定的一致性，不同的隔离级别，对于快照数据的定义页不同

| 隔离级别         | 说明                                     |
| ---------------- | ---------------------------------------- |
| READ COMMITED    | 非一致性读总是读取被锁定行的最新一份     |
| REPLEATABLE READ | 非一致性读总是读取事务开始时的行数据版本 |

#### 一致性锁定读

​	在默认的配置下，即事务的隔离级别为REPLEATABLE READ 模式下，InnoDB存储引擎的SELECT操作使用一致性非锁定读。但是在某些情况下，用户需要显示的对数据库读取操作进行加锁以保证数据逻辑一致性。

​	InnoDB存储引擎对于SELECT语句支持两种一致性的锁定读操作：

* SELECT 。。。FOR UPDATE

* SELECT 。。。LOCK IN SHARE MODE

  SELECT 。。。FOR UPDATE对读取的行记录加一个X锁。

  SELECT 。。。LOCK IN SHARE MODE 对读取的行记录加一个共享锁S

  注意：必须在一个事务中，当事务提交时，所也就释放了。因此，在使用上述两个语句时，务必加上，BEGIN,START TRANSACTION或SET AUTOCOMMIT = 0 

#### 自增长与锁

​	在InnoDB存储引擎的内存结构中，对每个含有自增长值得表都有一个自增长计数器。当对含有自增长的计数器的表进行插入时，这个计数器会被初始化，执行如下的语句来得到计数器的值

```sql
SELECT MAX(auto_inc_col) From t FOR UPDATE
```

​	插入操作会依据这个自增长的计数器值加1赋予自增长列。这种实现方式叫做：AUTO-INC Locking。这种锁采用一种特殊的表锁机制。为了提高插入的性能，锁不是在一个事务完成后才释放，而是在完成对自增长值插入的SQL语句后立即释放。

| 插入类型           | 说明                                                         |
| ------------------ | ------------------------------------------------------------ |
| insert-like        | 指所有的插入语句，如:INSERT,REPLACE,INSERT...SELECT,REPLACE...SELECT |
| simple inserts     | 指的是在插入前就确定插入行数的语句。                         |
| bulk inserts       | 指的是在插入前不能得到插入行数的语句                         |
| mixed-mode inserts | 指插入中有一部分是自增长，有一部分是确定的                   |

#### 外键与锁

​	在InnoDB存储引擎中，对于一个外键列，如果没有显式的加索引，InnoDB会自动对其加一个索引。

​	对于外键值得插入或更新，首先需要查询父表中的记录，即SELECT父表。但是对于父表的SELECT操作，不是使用一致性非锁定读的方式，因为这种方式会产生数据不一致的问题。因此，这时使用的是：SELECT ...LOCK IN SHARE MODE方式。

### 锁的算法

#### 行锁的3种算法

​	在InnoDB存储引擎有3中锁的算法，分别是：

* Record Lock ：单个行记录上的锁
* Gap Lock：间隙所，锁定一个范围，但不包含记录本身
* Next-Key Lock：Gap Lock + Record Lock ,锁定一个范围，并且锁定记录本身


​        Record Lock总是会去锁住索引记录，如果InnoDB存储引擎表建立的时候没有设置任何一个索引，那么这时InnoDB存储引擎或使用隐式的主键来进行锁定。

​	Next-Key Lock是结合了Gap Lock和Record Lock的一种锁定算法。在Next-Key Lock算法下，InnoDB对于行的查询都是采用这种锁定算法的。

​	例如：有一个索引有10,11,13和20这四个值，那么该索引可能被Next-Key Lock的区间为：

```shell
(-00,10]
(10,11]
(11,13]
(13,20]
(20,+00)

# Next-key 设计的目的是为了解决：Phantom Problem
```

​	当查询的索引含有唯一属性时，InnoDB存储引擎会对Next-key Lock进行优化，将其降级为Record Lock,即仅锁住索引本身。

![next-keyDemo](img\next-keyDemo.png)

```shell
# 表t中共有1，2,5三个值，在上面的例子中，回话A
# 首先：
# 对a=5 进行X锁定(由于a是主键且唯一，因此锁定的仅是5这个值，而不是锁(2,5】这个范围，这时Next-key Lock 降级为 Record Lock)
# 然后：
# 在回话B中，插入值4时，不会进行阻塞，可以立即插入并返回。提高了应用的并发性

# Next-key Lock降级为 Record Lock仅在查询的列为唯一索引的情况下
# 若是辅助索引，则情况会不同
```

```shell
CREATE TABLE z (a INT,b INT,PRIMARY KEY(a),key(b));
INSERT INTO z Select 1,1;
INSERT INTO z Select 3,1;
INSERT INTO z Select 5,3;
INSERT INTO z Select 7,6;
INSERT INTO z Select 10,8;

# 表z的列b是辅助索引，如在回话A中执行下面的SQL语句
SELECT * FROM z WHERE b=3 FOR UPDATE

# SQL语句通过索引b进行查询，因此其使用的是传统的Next-key Locking技术加锁，并且由于有2个索引，其需要分别进行锁定。对于索引a，其仅在 a=5的索引上加入Record Lock,而对于辅助索引，其加上的是Next-Key Lock，锁定范围是(1,3],
# 需要注意的是 InnoDB存储引擎还会对辅助索引下一个键值加上Gap Lock,即会对（3,6）进行加锁

# 若在回话B中运行下面的语句
SELECT * FROM z where a = 5 Lock in share mode;
insert into z select 4,2
insert into z select 6,5

# 第一个SQL语句，不能执行，因为在回话A中sql对a=5加上了X锁。因此执行会被阻塞
# 第二个SQL语句，主键插入4，没有问题，但是辅助索引值2，在锁定的访问（1,3）内，因此执行同样会被阻塞
# 第三个SQL语句，主键插入6，可以，5也不再范围(1,3)中，但是插入的值5在另一个锁定的范围(3,6)中，故同样需要等待。
```

#### 解决Phantom Problem

​	在默认的隔离级别下， 即Repeatable read下，InnoDB存储引擎采用Next-key Locking机制来避免Phantom problem问题(幻想问题)。

Phantom problem:在同一事物下，连续执行两次同样的SQL语句可能会导致不同的结果，第二次的SQL语句可能会返回之前不存在的行。



### 锁问题

​	通过锁定机制可以实现事物的隔离性，使得事务可以并发的工作。锁提高了并发，但是却会带来潜在的问题。不过因为事务隔离性的要求，锁只会带来三种问题。

#### 脏读

​	一个事务可以读取到另一个事务中未提交的数据，违反了数据库的隔离性。

### 不可重复读

​	不可重复读是指一个事务多次读取同一数据集合，在这个事务未结束时，另一个事务也访问该同一数据集合，并做了一些DML操作(INSERT,UPDATE,DELETE)。因此，在第一个事务中的两次读取数据之间，由于第二次事务的修改，第一次事务两次读取到的数据可能是不一样的。

| 名称       | 区别                         |
| ---------- | ---------------------------- |
| 不可重复读 | 读取到其他事务已经提交的数据 |
| 脏读       | 读取到其他事务未提交的数据   |

#### 丢失更新



​	

