# InnoDB存储引擎-读书笔记

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
* 页从缓冲池刷新回磁盘的操作，并不是在每次也发生更新时触发的。

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



#### 4、Checkpoint技术



## 关键特性

### 1、Insert Buffer

### 2、Double Write

### 3、adaptive Hash Index

### 4、异步IO

### 5、刷新邻接页





