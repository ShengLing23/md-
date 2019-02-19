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
#
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

#### 4、Page  Cleaner Thread

​	Page cleaner Thread在InnoDB 1.2.X版本中引入。作用是将之前版本中的脏页的刷新操作都放入到单独的线程中来完成。

### 内存

#### 1、缓冲池

#### 2、LRU List、Free List和Flush List

#### 3、重做日志缓冲

#### 4、Checkpoint技术



## 关键特性

### 1、Insert Buffer

### 2、Double Write

### 3、adaptive Hash Index

### 4、异步IO

### 5、刷新邻接页





