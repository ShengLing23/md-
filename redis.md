## redis

### 数据结构与对象

#### 简单动态字符串

+ redis 没有直接使用C的字符串表示，而是自己构建了SDS,作为默认字符串。

+ sds的定义

  ```c
  struct sdshdr{
      //buf中已经使用的字节数，sds所保持的字符串长度
      int len;
      //记录buf数组中未使用的字节数量
      int free;
      //字节数组，用于保存字符串
      char buf[];
  }；
  ```

+ SDS与C字符串的区别

  1、C字符串不记录自身长度，所以为了获取长度，必须遍历整个字符串，操作复杂度为O(N)

  ​      SDS的len属性，就是字符串长度，操作复杂度为O(1)

  2、C字符串不记录自身长度带来的另一个问题是容易造成缓冲区溢出

  ​	SDS完全杜绝了发生缓冲区溢出的可能性。当SDS的API进行修改时，首先会检查SDS的空间是否足够，不满足则会自动修改SDS的空间大小。

  3、减少了修改字符串的内存重分配此时

  + C字符串的长度和底层数组的长度存在一个N+1的关系，所以每次修改字符串，都会进行一次内存重新分配

  + SDS利用free，实现了空间预分配和毒性空间释放两个优化策列。

    + 空间预分配

      ​	当SDS的API对SDS进行修改时，不仅会为SDS分配修改必须的空间，还会为SDS分配额外的使用空间

      +  如果SDS修改后的len小于1MB，则为free分配和len属性同样大小的空间
      + 如果SDS修改后的len大于等于1MB，则为free分配1MB的空间 

    + 惰性空间释放

      ​	当SDS缩短时，并不立即使用内存重新分配来回收缩短出来的空间，而是使用free属性记录了下来。

  4、二进制安全

  + C字符串除了末尾外，不能包含空字符串，否则会被误认为是字符串的结尾。这个限制使得C字符串只能保存文本数据，而不能保存二进制数据
  + SDS使用len来确定是否时结束，所以可以保存特殊数据格式。

  5、兼容部分C字符串函数

####  链表

##### 链表和链表节点的实现

```c
//链表的节点
typedef struct listNode{
    //前置节点
    struct listNode *prev;
    //后置节点
    struct listNode *next;
    //节点的值
    void *value;
} listNode;

//链表
typedef struct list{
    //表头节点
    listNode *head;
    //表尾节点
    listNode *tail;
    //链表所包含的节点数
    unsigned long len;
    //节点值复制函数
    void *(*dup)(void *ptr);
    //节点值释放函数
    void *(*free)(void *ptr);
    //节点值比较函数
    int (*match)(void *ptr,void *key);
} list;
```

##### 特点

+ 双端：带有prev和next指针，获取前后节点的复杂度为O(1)
+ 无环
+ 带表头和表尾的指针
+ 带长度计数器
+ 多态，使用void*指针，可以保存不同类型的值

#### 字典

##### 字典的实现

redis的字典是使用哈希表作为底层实现的，一个哈希表中可以有多个哈希节点，而每个哈希节点保存了一个键值对

```c
//字典的结构
typedef struct dict{
    //类型特定函数
    dictType *type;
    //私有数据
    void *privdata;
    //哈希表
    dictht ht[2];
    //rehash索引，不进行时值为-1
    int rehashidx;
}
//类型特定函数
typedef struct dictType{
    //计算哈希值
    unsigned int (*hashFunction)(const void *key);
    //负责键的函数
    void *(*keyDup)(void *privdata,const void *key);
    //负责值的函数
    void *(*valDup)(void *privdata,const void *obj);
    //对比键的函数
    int (*keyCompare)(void *privdata,const void * key1,const void *key2);
    //销毁键的函数
    void (*keyDestructor)(void *privdata,void *key);
    //销毁值的函数
    void (*keyDestructor)(void *privdata,void *obj);
}
//哈希表
typedef struct dictht{
    //哈希表数组
    dictEntry **table;
    //哈希表大小
    unsigned long size;
    //哈希表大小掩码，用于计算哈希值，大小为size -1 
    unsigned long sizemask;
    //哈希表以有节点数
    unsigned long used;
} dictht;
//哈希节点
typedef struct dictEntry{
    void *key;
    union{
        void *val;
        uint64_t u64;
        int64_t s64;
    }
    struct dictEntry *next;
} v;
```

dict的ht属性是一个包含两个项的数组，每个项都是一个dictht哈希表，一般情况下使用ht[0],ht[1]会在对ht[0]进行rehash时使用。

##### 哈希算法

redis计算hash值和索引值的算法如下：

```shell
#计算，键key的哈希值
hash = dict->type->hashFunction(key)
#使用哈希表的sizemask属性和哈希值，计算出索引值
index = hash & dict->ht[x].sizemask;
```

##### 解决键冲突

redis使用链地址发来解决冲突，哈希节点有一个next的指针，多个哈希表节点可以组成链表。

被分配在同一个索引上的多个节点可以形成单项链表，解决键冲突

redis总是将新的节点添加到链表的头部

##### rehash

​	为了让哈希表的负载因子维持在一个合适的范围内，当哈希表保存的键值对数量太多或太少时，程序对哈希表的大小进行扩展或收缩

​	扩展和收缩由rehash（重新散列）来完成。

```shell
#1、为ht[1]哈希表分配空间，这个哈希表的空间大小取决于要执行的操作，以及ht[0]当前包含的键值对的数量
#	* 如果执行的是扩展操作，则ht[1]的数量是第一个大于等于ht[0].used*2的2的n次幂
#	* 如果执行的是收缩操作，则ht[1]的数量是第一个小于等于ht[0].used的2的n次幂
#2、将保存在ht[0]的所有键值对移送到ht[1]上面。rehash值得是重新计算键的哈希值和索引值，让后将键值对重新放置到ht[1]哈希表上面。
#3、当ht[0]的键值对都迁移到ht[1]之后，释放ht[0],将ht[1]设置为ht[0],并在ht[1]新建一个空白的哈希表。
```

#### 跳跃表

##### 实现

```c
//跳跃表
typedef struct zskiplist{
    //表头节点与表尾节点
    struct zskiplistNode * hearder,*tail;
    //表中节点的数量
    unsigned long length;
    //表中层数最大的节点的层数
} zskiplist;

//跳跃节点
typedef struct zskiplistNode{
    //层
    struct zskiplistLevel{
        //前进指针
        struct zskiplistNode *forward;
        //跨度
        unsigned int span;
    } level[];
    //后退指针
    struct zskiplistNode *backward;
    //分值
    double score;
    //成员对象
    robj *obj;
} zskiplistNode;
```

##### 重点

* 跳跃表是有序集合的底层实现之一
* 每个跳跃节点的层高都是1至32之间的随机数
* 在同一个跳跃表中，多个节点可以包含相同的分值，但是每个节点的对象必须是唯一的
* 跳跃表中的节点按照分值的大小进行排序，当分值相同时，节点按照成员对象进行排序

#### 压缩列表

##### 结构

| zlbytes | zltail | zllen | entry1 | entry2 | 。。。 | entryN | zlend |
| ------- | ------ | ----- | ------ | ------ | ------ | ------ | ----- |
|         |        |       |        |        |        |        |       |

| 属性    | 类型      | 长度  | 描述                                                         |
| ------- | --------- | ----- | ------------------------------------------------------------ |
| zlbytes | uint32_t  | 4字节 | 记录整个压缩列表占用的内存                                   |
| zltail  | uint32_t  | 4字节 | 记录压缩列表表尾节点距离压缩列表的起始地址有多少字节         |
| zllen   | uint_16_t | 2字节 | 记录了压缩列表的节点数量：小于uint16_max(65536)时，就是节点的真实数量，等于uint_16_max时，需要遍历才能计算出真实的节点数量 |
| entryX  | 列表节点  |       |                                                              |
| zlend   | uint8_t   | 1字节 | 特殊值0xFF,用于标记压缩列表的末端                            |

#### 对象

​	redis没有直接使用数据结构来实现键值对数据库，而是基于这些数据结构创建了一个对象系统。

​	这个对象系统包含：字符串对象，列表对象，哈希对象，集合对象和有序集合对象

​	redis的对象还实现了居于引用计数技术的内存回收机制。

##### 对象的类型和编码

​	redis使用对象来表示数据库中的键和值，每次创建一个键值对时，至少创建两个对象，一个对象用作键值对的键，另一个对象用作键值对的值。

```c
typedef struct redisObject{
    //类型
    unsigned type:4;
    //编码
    unsigned encoding:4;
    //指向底层实现数据结构的指针
    void *prt;
} robj;

/**redisObject中type的取值:
*    类型常量 | 对象的名称
*    REDIS_STRING | 字符串对象 
*    REDIS_LIST	  | 列表对象
*	 REDIS_HASH   | 哈希对象
*	 REDIS_SET	  | 集合对象
*    REDIS_ZSET	  | 有序集合对象
*/
```

###### 编码和底层实现

对象的prt指针指向对象的底层实现数据结构，而这些数据结构有对象的encoding属性决定，

| 编码常量                | 编码对应的底层数据结构     |
| ----------------------- | -------------------------- |
| REDIS_ENCODING_INT      | long类型的                 |
| REDIS_ENCODING_EMBSTR   | embstr编码的简单动态字符串 |
| REDIS_ENCODING_RAW      | 简单动态字符串             |
| REDIS_ENCODING_HT       | 字典                       |
| REDIS_ENCODING_LINKDLST | 双端列表                   |
| REDIS_ENCODING_ZIPLIST  | 压缩列表                   |
| REDIS_ENCODING_INTSET   | 整数集合                   |
| REDIS_ENCODING_SKIPLIST | 跳跃表和字典               |

###### 每种对象可以使用的编码对象

| 类型         | 编码                    |                            |
| ------------ | ----------------------- | -------------------------- |
| REDIS_STRING | REDIS_ENCODING_INT      | long类型的                 |
|              | REDIS_ENCODING_EMBSTR   | embstr编码的简单动态字符串 |
|              | REDIS_ENCODING_RAW      | 简单动态字符串             |
| REDIS_LIST   | REDIS_ENCODING_ZIPLIST  | 压缩列表                   |
|              | REDIS_ENCODING_LINKDLST | 双端列表                   |
| REDIS_HASH   | REDIS_ENCODING_ZIPLIST  | 压缩列表                   |
|              | REDIS_ENCODING_HT       | 字典                       |
| REDIS_SET    | REDIS_ENCODING_INTSET   | 整数集合                   |
|              | REDIS_ENCODING_HT       | 字典                       |
| REDIS_ZSET   | REDIS_ENCODING_ZIPLIST  | 压缩列表                   |
|              | REDIS_ENCODING_SKIPLIST | 跳跃表和字典               |

###### 优点

通过encoding属性来设定对象所使用的编码，而不是为特定的对象关联一种固定的编码，极大的提升了redis的灵活性和效率。

#### key操作

| 命令                  | 含义                        |
| --------------------- | --------------------------- |
| keys  *               | 查看当前库的所有键          |
| exists  <key>         | 查看 key 是否存在           |
| del  <key>            | 删除 key                    |
| expire <key> <second> | 设置key的过期时间           |
| ttl <key>             | 查看 key 还有多长的存活时间 |
| move <key>  <db>      | 把key移动到另一个库下       |
| dbsize                | 查看数据库KEY数量           |
| flushdb               | 情况当前库                  |
| flhshall              | 通杀所有库                  |



##### 字符串对象

​	字符串对象的编码可以是：int、raw或者embstr

* 如果字符串对象保存的时整数值，并且这个整数值可以用long类型来表示，那么字符串对象的编码设置为int
* 如果字符串对象保存的是一个字符串值，并且这个字符串值的长度大于32字节，那么字符串对象的编码设置为raw.
* 如果字符串对象保存的是一个字符串值，并且长度小于等于32字节，字符串对象的编码为embstr

###### 命令

| 命令        | int编码实现                                                | embstr编码实现                       | raw编码实现                                             |
| ----------- | ---------------------------------------------------------- | ------------------------------------ | ------------------------------------------------------- |
| SET         | 使用int编码保存                                            | 使用embstr编码保存                   | 使用编码保存                                            |
| GET         | 拷贝对象所保存的整数值，并将拷贝值转为字符串，返回给客户端 | 直接返回                             | 直接返回                                                |
| APPEND      | 将对象转为raw编码，然后按raw编码操作                       | 将对象转为raw编码，然后按raw编码操作 | 调用sdscatlen函数，将给定的字符串追加到现有字符串的末尾 |
| INCRBYFLOAT |                                                            |                                      |                                                         |
| INCRBY      | 对整数值进行加法操作                                       | 不能执行此命令                       | 不能执行此命令                                          |
| DECRBY      | 对整数值惊喜剑法操作                                       | 不能执行此命令                       | 不能执行此命令                                          |
| STRLEN      | 拷贝整数值，将其转为字符串，计算返回字符串的长度           | 调用sdslen函数，返回字符串长度       | 调用sdslen函数，返回字符串长度                          |
| SETRANGE    | 将对象转为raw编码，然后按raw编码操作                       | 将对象转为raw编码，然后按raw编码操作 | 将字符串特定索引上的值                                  |
| GETRANGE    | 拷贝整数值，将其转为字符串，然后返回字符串指定索引的字符   | 直接取出，并返回给定索引上的字符     | 直接取出，并返回给定索引上的字符                        |

##### 列表对象

​	列表对象的编码可以是：ziplist或者linkedlist

1、当列表对象可以同时满足如下两个条件时，列表对象可以使用ziplist编码

* 列表对象保存的所有字符串元素的长度都小于64字节
* 列表对象保存的元素数量小于512个

2、不满足以上两个条件的列表对象需要使用linkedlist编码

###### 命令

| 命令    | ziplist编码                                                  | linkedlist编码                                               |
| ------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| LPUSH   | 调用ziplistPush函数，将 新元素推入到压缩列表的表头           | 调用listAndNodeHead函数，将新元素推入到双端列表的表头        |
| RPUSH   | 调用ziplistPush函数，将 新元素推入到压缩列表的表尾           | 调用listAndNodeTail函数，将新元素推入到双端列表的表尾        |
| LPOP    | 调用ziplistIndex函数定位压缩列表的表头节点，在向用户返回节点所保存的元素之后，调用ziplistDelete函数删除表头节点。 | 调用listFirst函数定位双端链表的表头，在向用户返回节点所保存的元素之后,调用listDelNode函数删除表头节点 |
| RPOP    | 调用ziplistIndex函数定位压缩列表的表尾节点，在向用户返回节点所保存的元素之后，调用ziplistDelete函数删除表头节点。 | 调用listLast函数定位双端链表的表头，在向用户返回节点所保存的元素之后,调用listDelNode函数删除表头节点 |
| LINDEX  | 调用ziplistIndex函数定位压缩列表中的指定节点，然后返回节点所保存的元素 | d调用listIndex函数定位双端列表中指定节点，然后返回节点保存的元素 |
| LLEN    | 调用ziplistLen函数返回压缩列表的长度                         | 调用listLength函数返回双端列表的长度                         |
| lINSERT | 插入新节点到压缩列表的表头或者表尾，使用ziplistPush函数；插入新节点到压缩列表其他位置时，使用zipListInsert函数 | 调用listInsertNode函数，将新节点插入到双端列表的指定位置     |
| LREM    | 遍历压缩列表，并调用ziplistDelete函数删除包含了给定元素的节点 | 遍历双端列表，并调用listDelNode函数删除包含了给定元素的节点  |
| LTRIM   | 调用ziplistDeleteRange函数，删除压缩列表中所有不在指定索引范围内的节点 | 遍历双端列表，并调用listDelNode函数删除链表中所有不再指定索引范围内的节点 |
| LSET    | 调用ziplistDelete函数，先删除压缩列表只当索引上现有的节点，然后调用ziplistInsert函数，将一个包含给定元素的新节点插入到相同的索引上 | d调用listIndex函数，定位到双端列表只当索引上的节点，然后通过赋值操作更新节点的值 |

##### 哈希对象

 	哈希对象的编码可以时ziplist或者hashtable

1、当哈希对象可以同时满足以下两个条件时，哈希对象使用ziplist编码

- 哈希对象保存的所有字符串元素的长度都小于64字节
- 哈希对象保存的元素数量小于512个

2、不能满足以上两个条件的哈希对象使用hashtable编码

###### 命令

| 命令    | 含义 |
| ------- | ---- |
| HSET    |      |
| HGET    |      |
| HEXISTS |      |
| HDEL    |      |
| HLEN    |      |
| HGETALL |      |

##### 集合对象

​	集合对象的编码：intset 或者 hashtable

1、当集合对象满足下列条件时，对象使用intset编码

* 集合对象保存的所有元素时整数值
* 集合对象保存的元素数量不超过512个

2、不满足这两个条件的使用hashtable编码

###### 命令

| 命令        | 含义                                       |
| ----------- | ------------------------------------------ |
| SADD        | 添加                                       |
| SCARD       | 返回集合数量                               |
| SISMEMBER   | 查询元素释放存在于集合中                   |
| SMEMBERS    | 返回集合元素                               |
| SRANDMEMBER | 从集合中随机返回一个值                     |
| SPOP        | 从集合中随机返回一个值，并将其从集合中删除 |
| SREM        | 删除给定元素                               |

##### 有序集合对象

​	有序集合的编码可以时ziplist或者skiplist

1、当有序集合对象可以同时满足以下两个条件时，对象使用ziplist编码

* 有序集合保存的元素数量小于128
* 有序集合保存的所有元素成员的长度都小于64字节

2、不满足以上条件的有序集合对象，使用skiplist编码

###### 命令

| 命令      | 含义                                   |
| --------- | -------------------------------------- |
| ZADD      | 添加                                   |
| ZCARD     | 集合的数量                             |
| ZCOUNT    | 统计分值在给定范围内的节点数量         |
| ZRANGE    | 返回给定索引范围内的所有元素(从头开始) |
| ZREVRANGE | 返回给定索引范围内的所有元素(从尾开始) |
| ZRANK     | 元素排名（从头开始）                   |
| ZREVRANK  | 元素排名（从尾开始）                   |
| ZREW      | 删除给定元素                           |
| ZSCORE    | 获取给定元素的分值                     |



## 应用场景



