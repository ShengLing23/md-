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



