---
title: Redis
categories:
  - 数据库
tags:
  - Redis
abbrlink: 56288
date: 2022-05-13
cover : https://img-blog.csdnimg.cn/img_convert/738a04fea08a67bd5cbd112208fe8ef6.png

---



**Redis笔记撰写--------dgw**

# **Memcache与Redis：**



1. Memcache只支持单一的字符串数据类型，而Redis支持多种数据类型，集合什么的。
2. Memcache不能持久化到硬盘里面，而Redis可以持久化。
3. Memcache是多线程加锁的机制，而Redis是单线程加多路IO复用



# 数据结构

Redis中的key是字符串对象，而value可以是5种对象,字符串对象、列表对象、哈希对象、有序集合对象、集合对象



![img](../../images/Redis/XLXRTSWRU%7B4WYM9S%7BUK%5BO2.jpg)



 

| 字符串对象   | int        raw(SDS)  embstr           |
| ------------ | ------------------------------------- |
| 列表对象     | ziplist linkedlist(双端链表)          |
| 哈希对象     | ziplist hashtable(dict)               |
| 集合对象     | insert(整数数组)  hashtable（dict）   |
| 有序集合对象 | ziplist 或者 skiplist(Zset:跳表+字典) |

​    



## RedisDB主体数据结构

![img](../../images/Redis/3c386666e4e7638a07b230ba14b400fe.png)

## redisObject





该结构和保存数据有关的三个属性分别是**type**属性、**encoding**属性、**ptr**属性

**![img](../../images/Redis/58d3987af2af868dca965193fb27c464.png)**



- type，标识该对象是什么类型的对象（String 对象、 List 对象、Hash 对象、Set 对象和 Zset 对象）；
- encoding，标识该对象使用了哪种底层的数据结构；
- **ptr，指向底层数据结构的指针**。









## SDS



### SDS

Redis的string类型由SDS简单动态字符串实现

```c
struct sdshdr{
  //记录buf数组已使用字节的数量
  //等于SDS所保存字符串的长度
  int len;
  
  //记录buf数组中未使用字节的数量,也就是还可以使用几个字符，可以计算出剩余空间的大小
  int free;
  
  //字节数组，用于保存字符串
  char buf[];
  

}
```

**Redis5.0的SDS成员变量**

- **len，记录了字符串长度**。这样获取字符串长度的时候，只需要返回这个成员变量值就行，时间复杂度只需要 O（1）。
- **alloc，分配给字符数组的空间长度**。这样在修改字符串的时候，可以通过 `alloc - len` 计算出剩余的空间大小，可以用来判断空间是否满足修改需求，如果不满足的话，就会自动将 SDS 的空间扩展至执行修改所需的大小，然后才执行实际的修改操作，所以使用 SDS 既不需要手动修改 SDS 的空间大小，也不会出现前面所说的缓冲区溢出的问题(二进制安全)。
- **flags，用来表示不同类型的 SDS**。一共设计了 5 种类型，分别是 sdshdr5、sdshdr8、sdshdr16、sdshdr32 和 sdshdr64，后面在说明区别之处。
- **buf[]，字符数组，用来保存实际数据**。不仅可以保存字符串，也可以保存二进制数据。





### 二机制安全

因为 SDS 不需要用 “\0” 字符来标识字符串结尾了，而是**有个专门的 len 成员变量来记录长度，所以可存储包含 “\0” 的数据**。但是 SDS 为了兼容部分 C 语言标准库的函数， SDS 字符串结尾还是会加上 “\0” 字符。



因此， SDS 的 API 都是以处理二进制的方式来处理 SDS 存放在 buf[] 里的数据，程序不会对其中的数据做任何限制，数据写入的时候时什么样的，它被读取时就是什么样的。

通过使用二进制安全的 SDS，而不是 C 字符串，使得 Redis 不仅可以保存文本数据，也可以保存任意格式的二进制数据。

#### 不会发生缓冲区溢出

与C字符串不同，SDS的空间分配策略完全杜绝了发生缓冲区溢出的可能性：当SDS API需要对SDS进行修改时，API先会检查SDS的空间是否满足修改所需的要求，如果不满足的话，API会自动将SDS的空间扩展至执行修改所需的大小，然后才执行实际的修改操作，所以使用SDS既不需要手动修改SDS的空间大小，也不会出现缓冲区溢出问题。





### 内存重分配



C字符串每次增长或者缩短，程序都会对C字符串的数组进行一次内存重分配。



为了避免C字符串的这种缺陷，SDS通过未使用空间解除了字符串长度和底层数组长度之间的关联：在SDS中，buf数组的长度不一定就是字符数量加一，数组里面可以包含未使用的字节，而这些字节的数量就由SDS的free属性记录。



通过未使用空间，SDS实现了**空间预分配**和**惰性空间释放**两种优化策略

#### 空间预分配

**用于优化SDS字符串增长操作**

如果对SDS进行修改

SDS的长度小于1MB，那么程序分配和len属性同样大小的未使用空间，len和free值一样。

如果大于1MB，那么程序会分配1MB的未使用空间。



#### 惰性空间释放

**用于优化SDS字符串缩短操作**

当SDS的API需要缩短SDS所保存的字符串时，程序并不立即使用内存重分配来回收缩短后多出来的字节，而是使用free属性将这些字节的数量记录起来，并等待将来使用。也就是说，缩短后的字符串不用的空间，不会立即释放，而是保留在free等以后使用





由SDS数据结构实现string类型

 

## 字典

Redis的字典使用**哈希表**作为底层实现，一个哈希表里面可以由多个哈希表结点，而每个哈希表结点就保存了字典中的一个键值对。

### dict

```c
typedef struct dict {
    
    //类型特定函数
    dictType *type;
    
    //私有数据
    void *privdata;
    
    …
    //两个Hash表，交替使用，用于rehash操作
    dictht ht[2]; 
   
    
    //rehash索引
    //当rehash不在进行时，值为-1
    int trehashidx;
    
} dict;
```





下面是**哈希表**结构体

```c
typedef struct dictht {
    //哈希表数组
    dictEntry **table;
    //哈希表大小
    unsigned long size;  
    //哈希表大小掩码，用于计算索引值
    //总是等于size - 1
    unsigned long sizemask;
    //该哈希表已有的节点数量
    unsigned long used;
} dictht;
```





哈希表结点由dictEntry 实现

```c
typedef struct dictEntry {
    //键值对中的键
    void *key;
  
    //键值对中的值
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    //指向下一个哈希表节点，形成链表
    struct dictEntry *next;
} dictEntry;
```



dictEntry 结构里不仅包含指向键和值的指针，还包含了指向下一个哈希表节点的指针，这个指针可以将多个哈希值相同的键值对链接起来，以此来解决**哈希冲突**的问题，这就是链式哈希。



**哈希算法**：当要将一个新的键值对添加到字典里面时，先根据键计算出哈希值与索引值，然后再根据索引值，将包含键值对的哈希表结点放到哈希表数组的指定索引上面





**哈希冲突**：头插



### rehash



**负载因子计算**

负载因子 = 哈希表已经保存结点数量 / 哈希表大小。

load_factor = ht[0].used / ht[0].size;



**随着操作的不断进行，哈希表保存的键值对会逐渐增多或者减少，为了让哈希表的负载因子（load factor）维持再一个合理的范围之内，当哈希表保存的键值对数量太多或者太少时，程序需要对哈希表的大小进行相应的扩展或者收缩，也就是rehash(重新散列)**



**步骤（很重要）**：



dict里面有两个哈希表ht[0]和ht[1]



**1**.为字典的ht[1]哈希表分配空间，这个哈希表的空间大小取决于要执行的操作以及ht[0]当前包含的**键值对数量**（也即是ht[0].used的值）：

​                如果是扩展操作，那么ht[1]的大小为第一个大于等于**ht[0].used * 2** 的 "**2的n次方幂**"；

​                如果是收缩操作，那么ht[1]的大小为第一个大于等于  **ht[0].used**  的  **2的n次方幂**

**2**.将保存在ht[0]的所有键值对rehash到ht[1]上面：rehash指的是重新计算键的哈希值和索引值，然后将键值对放置到ht[1]哈希表的指定位置上。



**3**.当ht[0]包含的所有键值对都迁移到了ht[1]之后，释放ht[0],将ht[1]设置为ht[0],并在ht[1]新创建一个空白哈希表，为下一次rehash做准备





### rehash 触发条件

**扩展**



- 服务器目前没有在执行BGSAVE命令或者BGREWRITEAOF命令，并且哈希表的负载因子大于等于1
- 服务器目前在执行BGSAVE命令或者BGREWRITEAOF命令，并且哈希表的负载因子大于等于5



**收缩**

当哈希表的负载因子小于0.1时，程序开始自动对哈希表进行收缩



**负载因子计算**

负载因子 = 哈希表已经保存结点数量 / 哈希表大小。

load_factor = ht[0].used / ht[0].size;



### 渐进式rehash

扩展或收缩哈希表需要将ht[0]的所有键值对rehash到ht[1]里面，但是这个rehash并不是一次性、集中式地完成的，而是分多次、渐进式地完成的。



**步骤（很重要）**

- 为ht[1]分配空间，让字典同时持有ht[0]和ht[1]两个哈希表

  

- 在字典中（dict）维持一个索引计数器变量rehashidx,并将它的值设置为0（默认是-1,表示没开始），表示rehash工作正式开始

  

- 在rehash进行期间，每次对字典执行添加、删除、查找或者更新操作时，程序除了执行指定的操作以外，还会顺带将ht[0]哈希表在rehashidx索引上（哈希桶）的所有键值对rehash到ht[1]，当rehash工作完成之后，程序将rehashidx的值加1.

  

- 随着字典操作的不断进行，最终在某个时间点上ht[0]的所有键值对都会rehash到ht[1],这时将rehashidx值设置为-1，表示rehash操作已经完成。

- 最后释放ht[0],将ht[1]设置为ht[0]



**渐进式rehash好处**：采用分而治之的方式，**避免了集中rehash而带来的庞大计算量。**





**渐进式rehash执行期间，执行的哈希表操作**

要在字典里面查找有一个键的话，先在ht[0]里面找，如果没找到的话，就会继续到ht[1]里面进行查找。（**先在原哈希表中找，如果没找到再在rehash后的哈希表中找**）

在渐进式rehash期间，新添加的键值对会保存到ht[1]里面，保证ht[0]只减不增。（在渐进式rehash期间，新添加的键值对会保存到rehash后的哈希表中）









## 链表



Redis 的 List 对象的底层实现之一就是链表。C 语言本身没有链表这个数据结构的，所以 Redis 自己设计了一个链表数据结构。



链表结点



```c
typedef struct listNode {
    //前置节点
    struct listNode *prev;
    //后置节点
    struct listNode *next;
    //节点的值
    void *value;
} listNode;
```



有前置节点和后置节点，可以看的出，这个是一个双向链表。



![img](../../images/Redis/4fecbf7f63c73ec284a4821e0bfe2843.png)



不过，Redis 在 listNode 结构体基础上又封装了 list 这个数据结构，这样操作起来会更方便，链表结构如下：



```c
typedef struct list {
    //链表头节点
    listNode *head;
    //链表尾节点
    listNode *tail;
    //节点值复制函数
    void *(*dup)(void *ptr);
    //节点值释放函数
    void (*free)(void *ptr);
    //节点值比较函数
    int (*match)(void *ptr, void *key);
    //链表节点数量
    unsigned long len;
} list;
```



![img](../../images/Redis/cadf797496816eb343a19c2451437f1e.png)





### **优点**

- listNode 链表节点的结构里带有 prev 和 next 指针，**获取某个节点的前置节点或后置节点的时间复杂度只需O(1)，而且这两个指针都可以指向 NULL，所以链表是无环链表**；
- list 结构因为提供了表头指针 head 和表尾节点 tail，所以**获取链表的表头节点和表尾节点的时间复杂度只需O(1)**；
- list 结构因为提供了链表节点数量 len，所以**获取链表中的节点数量的时间复杂度只需O(1)**；
- listNode 链表节使用 void* 指针保存节点值，并且可以通过 list 结构的 dup、free、match 函数指针为节点设置该节点类型特定的函数，因此**链表节点可以保存各种不同类型的值**；

### **缺点**

- 链表每个节点之间的内存都是不连续的，意味着**无法很好利用 CPU 缓存**。能很好利用 CPU 缓存的数据结构就是数组，因为数组的内存是连续的，这样就可以充分利用 CPU 缓存来加速访问。
- 还有一点，保存一个链表节点的值都需要一个链表节点结构头的分配，**内存开销较大**。



### **总结**

链表被广泛用于实现Redis的各种功能，比如列表键、发布与订阅、慢查询、监视器等。



**Redis的链表实现是双端链表**



Redis 3.0 的 List 对象在数据量比较少的情况下，会采用「压缩列表」作为底层数据结构的实现，它的优势是节省内存空间，并且是内存紧凑型的数据结构。

不过，压缩列表存在性能问题（具体什么问题，下面会说），所以 Redis 在 3.2 版本设计了新的数据结构 quicklist，并将 List 对象的底层数据结构改由 quicklist 实现。

然后在 Redis 5.0 设计了新的数据结构 listpack，沿用了压缩列表紧凑型的内存布局，最终在最新的 Redis 版本，将 Hash 对象和 Zset 对象的底层数据结构实现之一的压缩列表，替换成由 listpack 实现。



## 压缩列表(ziplist)

压缩列表（ziplist）是列表键和哈希键的底层实现之一。当一个列表键（list）只包含少量列表项，并且每个列表项要么就是小整数值，要么就是长度笔记短的字符串，那么Redis就会使用压缩列表来做列表键的底层实现。另外，当一个哈希键只包含少了键值对，并且每个键值对的键和值要么就是小整数值，要么就是长度比较短的字符串，那么Redis就会使用压缩列表来做哈希键的底层实现。





![image-20220509212718139](../../images/Redis/image-20220509212718139.png)

- zlbytes:4个字节，整个压缩列表占用的内存字节数
- zltail：尾节点指针
- zllen,压缩列表所包含的结点数目
- zlend:特殊值，标记压缩列表的末端





**结点：**

压缩列表是Redis为了节约内存而开发的，是由一系列特殊编码的**连续内存块组成的顺序型数据结构**。一个压缩列表可以包含**任意多个结点**，每个结点可以保存一个字节数组或者一个整型值，



### 连锁更新



每个结点有一个previous_entry_length属性，记录了压缩列表前一个结点的长度，这个previous_entry_length属性的长度可以是1字节或者5字节



**压缩列表新增某个元素或修改某个元素时，如果空间不不够，压缩列表占用的内存空间就需要重新分配。而当新插入的元素较大时，可能会导致后续元素的 prevlen 占用空间都发生变化，从而引起「连锁更新」问题，导致每个元素的空间都要重新分配，造成访问压缩列表性能的下降**。



压缩列表节点的 prevlen 属性会根据前一个节点的长度进行不同的空间大小分配：

- 如果前一个**节点的长度小于 254 字节**，那么 prevlen 属性需要用 **1 字节的空间**来保存这个长度值；

- 如果前一个**节点的长度大于等于 254 字节**，那么 prevlen 属性需要用 **5 字节的空间**来保存这个长度值；

  





### 总结：

- ​        压缩列表是Redis为了节约内存而开发的，是由一系列特殊编码的连续内存块组成的顺序型数据结构。
- ​       压缩列表（ziplist）是**列表键**和**哈希键**的底层实现之一
- ​       一个压缩列表可以包含**任意多个结点**，每个结点可以保存一个字节数组或者一个整型值
- ​       添加新结点到压缩列表，或者从压缩列表中删除结点，可能会引发连锁更新，但这种操作出现的机率不高







## 跳表



跳跃表（skiplist）是一种有序数据结构，它通过在每个结点维持多个指向其他结点的指针，从而达到快速访问结点的目的。

支持平均O（logN）,最坏O(N)查找复杂度。跳跃表的实现比平衡树简单，所有不少程序都使用跳表来代替平衡树。





**Redis只在两个地方使用到了跳表**：1.一个是实现有序结合键（Zset）2.另一个是在集群结点中使用作为内部数据结构。



**zskiplist（表结构）**

```c
typedef struct zskiplist {
    
    //header指向跳表的表头结点，tail指向跳表的表尾结点
    struct zskiplistNode *header, *tail;
    
    //跳表的长度，目前包含结点的长度（表头结点不计）
    unsigned long length;
    
    //记录目前跳表内，层数最大的那个结点的层数，（表头的层数不计算）
    int level;
} zskiplist;
```



书上表头结点有32层





**zskiplistNode**(跳表结点)

```c
typedef struct zskiplistNode {
    //Zset 对象的元素值
    sds ele;
    //元素权重值
    double score;
    //后向指针
    struct zskiplistNode *backward;
  
    //节点的level数组，保存每层上的前向指针和跨度
    struct zskiplistLevel {
        //前进指针，在查询的时候不断前进
        struct zskiplistNode *forward;
        //跨度
        unsigned long span;
    } level[];
    
    
} zskiplistNode;
```



**level（层）**

一般来说，结点层的数量越多，访问其他结点的速度就越快。，每次创建一个新跳表结点的时候，程序都根据幂次定律随机生成一个介于1和32之间的值作为level数组的大小，这个大小就是层的高度



**前进指针forward**

每层都有，用于从表头向表尾方向访问结点。



**跨度**

层的跨度(level[i].span)用于计算两个结点之间的距离。



- 两个结点跨度越大，它们相距的就越远。
- 指向NULL的所有前进指针的跨度都为0，因为它们没有连向任何结点



**排位**

跨度是用来计算排位的（rank）：在查找某个结点的过程中，将沿途访问过的所有层的跨度累加起来，得到的结果就是目标结点在跳表中的排位







**分值和成员**

分值是一个double类型的数据

成员对象（obj属性）是一个指针，指向一个字符串对象，也就是SDS.

每个结点的成员对象是唯一的，但分值确实可以不唯一的，分值相同的结点按照成员对象的字典序排列



### 查询过程

查找一个跳表节点的过程时，跳表会从头节点的最高层开始，逐一遍历每一层。在遍历某一层的跳表节点时，会用跳表节点中的 SDS 类型的**元素和元素的权重**来进行判断，共有两个判断条件：

- 如果当前节点的权重「**小于**」**要查找的权重**时，跳表就会访问该层上的下一个节点。
- 如果当前节点的权重「**等于**」**要查找的权重时**，并且当前节点的 SDS 类型数据「小于」要查找的数据时，跳表就会访问该层上的下一个节点。

如果上面两个条件都不满足，或者下一个节点为空时（也就是当前结点分值大于要查找的分值时），跳表就会使用目前遍历到的节点的 level 数组里的下一层指针，然后沿着下一层指针继续查找，这就相当于跳到了下一层接着查找。

**例1**

![img](../../images/Redis/47275977397a5ab24dc64000c04e9427.png)

如果要查找「元素：abcd，权重：4」的节点，查找的过程是这样的：

- 先从头节点的最高层开始，L2 指向了「元素：abc，权重：3」节点，这个节点的权重比要查找节点的小，所以要访问该层上的下一个节点；
- 但是该层上的下一个节点是空节点，于是就会跳到「元素：abc，权重：3」节点的下一层去找，也就是 leve[1];
- 「元素：abc，权重：3」节点的 leve[1] 的下一个指针指向了「元素：abcde，权重：4」的节点，然后将其和要查找的节点比较。虽然「元素：abcde，权重：4」的节点的权重和要查找的权重相同，但是当前节点的 SDS 类型数据「大于」要查找的数据，所以会继续跳到「元素：abc，权重：3」节点的下一层去找，也就是 leve[0]；
- 「元素：abc，权重：3」节点的 leve[0] 的下一个指针指向了「元素：abcd，权重：4」的节点，该节点正是要查找的节点，查询结束。



**例2**

![img](data:image/jpeg;base64,/9j/4QC7RXhpZgAATU0AKgAAAAgABgEaAAUAAAABAAAAVAEbAAUAAAABAAAAXQEoAAMAAAABAAIAAAESAAMAAAABAAEAAAExAAIAAAATAAAAZodpAAQAAAABAAAAfQAAAAABLAAAAAEAAAABLAAAAAEAUG9sYXJyIFBob3RvIEVkaXRvcgAAAAAABKACAAQAAAABAAAEdqADAAQAAAABAAAB5qABAAMAAAABAAEAAJAAAAcAAAAEMDIzMQAAAAD/2wCEAAMCAgMCAgMDAwMEAwMEBQgFBQQEBQoHBwYIDAoMDAsKCwsNDhIQDQ4RDgsLEBYQERMUFRUVDA8XGBYUGBIUFRQBAwQEBQQFCQUFCRQNCw0UFBQUFBQUFBQUFBQUFBQUFBQUFBQUFBQUFBQUFBQUFBQUFBQUFBQUFBQUFBQUFBQUFP/AABEIAeYEdgMBEQACEQEDEQH/xAGiAAABBQEBAQEBAQAAAAAAAAAAAQIDBAUGBwgJCgsQAAIBAwMCBAMFBQQEAAABfQECAwAEEQUSITFBBhNRYQcicRQygZGhCCNCscEVUtHwJDNicoIJChYXGBkaJSYnKCkqNDU2Nzg5OkNERUZHSElKU1RVVldYWVpjZGVmZ2hpanN0dXZ3eHl6g4SFhoeIiYqSk5SVlpeYmZqio6Slpqeoqaqys7S1tre4ubrCw8TFxsfIycrS09TV1tfY2drh4uPk5ebn6Onq8fLz9PX29/j5+gEAAwEBAQEBAQEBAQAAAAAAAAECAwQFBgcICQoLEQACAQIEBAMEBwUEBAABAncAAQIDEQQFITEGEkFRB2FxEyIygQgUQpGhscEJIzNS8BVictEKFiQ04SXxFxgZGiYnKCkqNTY3ODk6Q0RFRkdISUpTVFVWV1hZWmNkZWZnaGlqc3R1dnd4eXqCg4SFhoeIiYqSk5SVlpeYmZqio6Slpqeoqaqys7S1tre4ubrCw8TFxsfIycrS09TV1tfY2dri4+Tl5ufo6ery8/T19vf4+fr/2gAMAwEAAhEDEQA/AP01rAoKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKAGu6xqWYhVAySTgCk2krsaTbsgRxIoZTkHoaUZKSuhtNOzHVRIUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFADWRXxuUNg5GaTSe5SbWw6mSFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFACdKADI9aAFoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgBGYIMsQB6mgDlfEPxI0jQGMZl+1XH/ADzh5x9T2qXJIqx5zqvxd1q8mb7KyWcPZQoZvzNZ8zKsjnr3xjrWoxhJ9SuHQHOA+3+VK7HYrjxFqihQNRusL0HnNgfrRcLFseNdcUIBqlxhOnz/AOc0XYWNbSfirrmnOPOmW9j7rKOfzFNSYrI9b8KeMLLxXZiSBhHOv+sgY/Mv/wBatE7kNWN6qEFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFAGH4q8WWnhOx8+4y8jcRxL1Y0m7DSueK+JfH2qeJXKyS/Z7YH5YYuB+J71i22WlY5xwVYgnJ+tIYlAwoAKACgAoAuaTq1zol9Hd2khjlQ59j7GjYR6joHxlhuHSHU7byCQB50ZyCfcdq0Uu5PKejWV7BqNslxbSrNC4yrqcg1ZJPTEFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQ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KACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKACgAoAKAMXxX4Xt/FemG1nJjYHckqjlTSauNOx4t4j+HmreHjJI0JuLVf+W0XPHuO1ZNNFpnMEEdRipGFAxyxu3RSfoKBD1tJ3xthkbJwMKaAN3TPh/ruq7GisXjjbHzynaMevNOzFdHoWi/Bqwt4kbUZ5LibqVjO1Pp6mrUe4uY7TTfDum6QP8ARLKGA4wWVRk/jV2SJuaNMQtABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAhAYEEAg9jQBlXnhTSNQz5+nwOT32AGlZDuV4/AugxOjLpkGU6ZXNKyC7NWDTLS2TbFbRRr6KgFOwEot4gOIkHOfuigB9MQtABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFABQAUAFAH/9k=)

如果我们现在要查找某个结点，比如 16。我们可以先在索引层遍历，当遍历到索引层中值为 13 的结点时，我们发现下一个结点是 17，那要查找的结点 16 肯定就在这两个结点之间。然后我们通过索引层结点的 down 指针，下降到原始链表这一层，继续遍历。这个时候，我们只需要再遍历 2 个结点，就可以找到值等于 16 的这个结点了。这样，原来如果要查找 16，需要遍历 10 个结点，现在只需要遍历 7 个结点。





### 跳表实现

java实现







### 总结





除了上面的，跳表是有序集合Zset的底层实现

**跳表的相邻两层的节点数量最理想的比例是 2:1，查找复杂度可以降低到 O(logN)**。



![image-20220509213827425](../../images/Redis/image-20220509213827425.png)



## 整数集合

整数集合是 Set 对象的底层实现之一。当一个 Set 对象只包含整数值元素，并且元素数量不大时，就会使用整数集这个数据结构作为底层实现。



**整数集合本质上是一块连续内存空间，它的结构定义如下：**

```c
typedef struct intset {
    //编码方式
    uint32_t encoding;
    //集合包含的元素数量
    uint32_t length;
    //保存元素的数组
    int8_t contents[];
} intset;
```





## bitMap

Bit-map的基本思想就是用一个bit位来标记某个元素对应的Value，而Key即是该元素。由于采用了Bit为单位来存储数据，因此在存储空间方面，可以大大节省。（PS：划重点 **节省存储空间**）



**这个东西还得看别人博客**

- https://www.cnblogs.com/cjsblog/p/11613708.html

  

- https://mp.weixin.qq.com/s/LavkCpqMTled_1m9CpJQ6w




上面这两篇博客写的很好



> 如果我们需要记录某一用户在一年中每天是否有登录我们的系统这一需求该如何完成呢？如果使用KV存储，每个用户需要记录365个，当用户量上亿时，这所需要的存储空间是惊人的。
>
> **Redis 为我们提供了位图这一数据结构，每个用户每天的登录记录只占据一位，365天就是365位，仅仅需要46字节就可存储，极大地节约了存储空间。**
>
> ![图片](../../images/Redis/640.png)
>
> **位图数据结构其实并不是一个全新的玩意，我们可以简单的认为就是个数组，只是里面的内容只能为0或1而已(二进制位数组)。**







### 基本原理

BitMap 的基本原理就是用一个 bit 来标记某个元素对应的 Value，而 Key 即是该元素。由于采用一 个bit 来存储一个数据，因此可以大大的节省空间。

我们通过一个具体的例子来说明 BitMap 的原理，假设我们要对 0-31 内的 3 个元素 (10, 17,28) 排序，那么我们就可以采用 BitMap 方法（假设这些元素没有重复）。

如下图，要表示 32 个数，我们就只需要 32 个 bit（4Bytes），首先我们开辟 4Byte 的空间，将这些空间的所有 bit 位都置为 0。

![img](../../images/Redis/1507794095528_2851_1507794091109.png)

然后，我们要添加(10, 17,28) 这三个数到 BitMap 中，需要的操作就是在相应的位置上将0置为1即可。如下图，比如现在要插入 10 这个元素，只需要将蓝色的那一位变为1即可。

![img](../../images/Redis/1507794107931_1343_1507794103528.png)

将这些数据插入后，假设我们想对数据进行排序或者检索数据是否存在，就可以依次遍历这个数据结构，碰到位为 1 的情况，就当这个数据存在。

#### 字符串映射

BitMap 也可以用来表述字符串类型的数据，但是需要有一层Hash映射，如下图，通过一层映射关系，可以表述字符串是否存在。

![img](../../images/Redis/1507794122751_7136_1507794118245.png)

当然这种方式会有数据碰撞的问题，但可以通过 Bloom Filter 做一些优化。





### 场景

大量数据的快速排序、查找、去重



**快速排序**

假设我们要对0-7内的5个元素(4,7,2,5,3)排序（这里假设这些元素没有重复）,我们就可以采用Bit-map的方法来达到排序的目的。

要表示8个数，我们就只需要8个Bit（1Bytes），首先我们开辟1Byte的空间，将这些空间的所有Bit位都置为0，然后将对应位置为1。

最后，遍历一遍Bit区域，将该位是一的位的编号输出（2，3，4，5，7），这样就达到了排序的目的，时间复杂度O(n)。

优点：

- 运算效率高，不需要进行比较和移位；
- 占用内存少，比如N=10000000；只需占用内存为N/8=1250000Byte=1.25M

缺点：

- **所有的数据不能重复。即不可对重复的数据进行排序和查找。**

- 只有当数据比较密集时才有优势

  

**快速去重**

20亿个整数中找出不重复的整数的个数，内存不足以容纳这20亿个整数。 

首先，根据“内存空间不足以容纳这05亿个整数”我们可以快速的联想到Bit-map。下边关键的问题就是怎么设计我们的Bit-map来表示这20亿个数字的状态了。其实这个问题很简单，一个数字的状态只有三种，分别为不存在，只有一个，有重复。因此，我们只需要2bits就可以对一个数字的状态进行存储了，假设我们设定一个数字不存在为00，存在一次01，存在两次及其以上为11。那我们大概需要存储空间2G左右。

接下来的任务就是把这20亿个数字放进去（存储），如果对应的状态位为00，则将其变为01，表示存在一次；如果对应的状态位为01，则将其变为11，表示已经有一个了，即出现多次；如果为11，则对应的状态位保持不变，仍表示出现多次。

最后，统计状态位为01的个数，就得到了不重复的数字个数，时间复杂度为O(n)。



**快速查找**

这就是我们前面所说的了，int数组中的一个元素是4字节占32位，那么除以32就知道元素的下标，对32求余数（%32）就知道它在哪一位，如果该位是1，则表示存在。



**上亿级别系统做日活统计**

请求到的用户进行hash计算且除以预算好的值进行进行取余运算，统计bit位是1的数据量也就是用户的日活量



**用户的回访统计，将两天的bitMap进行and运算**

将两天的日活量的数据进行取and运算，然后是1的也就是回访的用户量、



### 代码

```java
public class BitMap { // Java 中 char 类型占 16bit，也即是 2 个字节
  private char[] bytes;
  private int nbits;
  
  public BitMap(int nbits) {
    this.nbits = nbits;
    this.bytes = new char[nbits/16+1];
  }
 
  public void set(int k) {
    if (k > nbits) return;
    int byteIndex = k / 16;
    int bitIndex = k % 16;
    bytes[byteIndex] |= (1 << bitIndex);
  }
 
  public boolean get(int k) {
    if (k > nbits) return false;
    int byteIndex = k / 16;
    int bitIndex = k % 16;
    return (bytes[byteIndex] & (1 << bitIndex)) != 0;
  }
}
```



### 



### BitMap去重

https://www.cnblogs.com/z-sm/p/6238977.html

**40亿QQ号去重**



一个字节可以记录8个数是否存在(类似于计数排序)，将QQ号对应的offset的值设置为1表示此数存在，遍历完40亿个QQ号后直接统计`BITMAP`上值为1的offset即可完成QQ号的去重。

> **如果是对40亿个QQ号进行排序也是可以用位图完成的哦~一样的思想🐂**









### **位图实战**



https://mp.weixin.qq.com/s/LavkCpqMTled_1m9CpJQ6w









# 过期键的删除

## 主动删除



### 定时删除

在设置键的过期时间的同时，创建一个定时器，让定时器在键的过期时间来临时，立即执行对键的删除操作



**优缺点：**对CPU不友好。



### 定期删除



每隔一段时间，程序就对数据库进行一次检查，删除里面的过期键，至于要删除多少过期键，以及要检查多少个数据库，由算法决定



**优缺点：**折中



## 被动删除



### 惰性删除

放任键过期不管，但是每次从键空间获取键时，都检查取得的键是否过期，如果过期的话，就删除改键，如果没有过期，就返回该键。



**优缺点：**对内存不友好







Redis服务器实际采用的是惰性删除和定期删除两种策略。





![img](../../images/Redis/04bdd13b760016ec3b30f4b02e133df6.jpg)



**a) 针对设置了过期时间的key做处理：**

1. volatile-ttl：在筛选时，会针对设置了过期时间的键值对，根据过期时间的先后进行删除，越早过期的越先被删除。
2. volatile-random：就像它的名称一样，在设置了过期时间的键值对中，进行随机删除。
3. volatile-lru：会使用 LRU 算法筛选设置了过期时间的键值对删除。
4. volatile-lfu：会使用 LFU 算法筛选设置了过期时间的键值对删除。

**b) 针对所有的key做处理：**

1. allkeys-random：从所有键值对中随机选择并删除数据。
2. allkeys-lru：使用 LRU 算法在所有数据中进行筛选删除。
3. allkeys-lfu：使用 LFU 算法在所有数据中进行筛选删除。

**c) 不处理：**

1. noeviction：不会剔除任何数据，拒绝所有写入操作并返回客户端错误信息"(error) OOM command not allowed when used memory"，此时Redis只响应读操作。







# Redis网络

前言：本模块相当于外传与引申，关于epoll IO多路复用要看下一章节[单线程和epoll多路复用高性能]



## select

select 机制中的一个重要函数就是 select 函数。对于 select 函数来说，它的参数包括监听 的文件描述符数量__nfds、被监听描述符的三个集合**readfds**、__writefds和 __**exceptfds**，以及监听时阻塞等待的超时时长__timeout**。下面的代码显示了 **select 函数的原型，你可以看下。



```c
1 int select (int __nfds, fd_set *__readfds, fd_set *__writefds, fd_set *__excep
```





这里你需要注意的是，Linux 针对每一个套接字都会有一个文件描述符，也就是一个非负整 数，用来唯一标识该套接字。所以，在多路复用机制的函数中，Linux 通常会用文件描述符 作为参数。有了文件描述符，函数也就能找到对应的套接字，进而进行监听、读写等操 作。







所以，select 函数的参数__readfds、__writefds和__exceptfds表示的是，被监听 描述符的集合，其实就是被监听套接字的集合。那么，为什么会有三个集合呢？ 这就和我刚才提出的第一个问题相关，也就是多路复用机制会监听哪些事件。select 函数 使用三个集合，表示监听的三类事件，分别是读数据事件（对应__readfds集合）、写数 据事件（对应__writefds集合）和异常事件（对应__exceptfds集合）。









### select网络通信

首先，我们在调用 select 函数前，可以先创建好传递给 select 函数的描述符集合，然后再 创建监听套接字。而为了让创建的监听套接字能被 select 函数监控，我们需要把这个套接 字的描述符加入到创建好的描述符集合中。 然后，我们就可以调用 select 函数，并把创建好的描述符集合作为参数传递给 select 函 数。**程序在调用 select 函数后，会发生阻塞。而当 select 函数检测到有描述符就绪后，就 会结束阻塞，并返回就绪的文件描述符个数。**



然后，我们就可以调用 select 函数，并把创建好的描述符集合作为参数传递给 select 函 数。程序在调用 select 函数后，会发生阻塞。而当 select 函数检测到有描述符就绪后，就 会结束阻塞，并返回就绪的文件描述符个数。



![image-20220814142905750](../../images/Redis/image-20220814142905750.png)







### select设计不足



1. 首先，select 函数对单个进程能监听的文件描述符数量是有限制的，它能监听的文件描 述符个数由 __FD_SETSIZE 决定，默认值是 1024。 

   

2. 其次，当 select 函数返回后，我们需要遍历描述符集合，才能找到具体是哪些描述符就 绪了。这个遍历过程会产生一定开销，从而降低程序的性能。（select返回的是所有的文件描述符集合）



## poll

为了解决 select 函数受限于 1024 个文件描述符的不足，poll 函数对此做了改进。



```c
1 int poll (struct pollfd *__fds, nfds_t __nfds, int __timeout);
```



和 select 函数相比，poll 函数的改进之处主要就在于，它允许一次监听超过 1024 个文件描述符。但是当调用了 poll 函数后，我们仍然需要遍历每个文件描述符，检测该描 述符是否就绪，然后再进行处理。





那么，有没有办法可以避免遍历每个描述符呢？这就是我接下来向你介绍的 epoll 机制。

## epoll



对于 epoll 机制来说，我们则需要先调用 epoll_create 函数，创建一个 epoll 实例





这个 epoll 实例内部维护了两个结构，分别是记录要监听的文件描述符和已经就绪的文件 描述符，而对于已经就绪的文件描述符来说，它们会被返回给用户程序进行处理。





在创建了 epoll 实例后，我们需要再使用 epoll_ctl 函数，给被监听的文件描述符添加监听 事件类型，以及使用 epoll_wait 函数获取就绪的文件描述符。









![image-20220814145359791](../../images/Redis/image-20220814145359791.png)





















## 三者总结

![image-20220814145836304](../../images/Redis/image-20220814145836304.png)













##  Reactor 模型

**具体看文章详解**



问题

**Redis 的网络框架是 实现了 Reactor 模型吗**





**介绍：Reactor 模型是高性能网络系统实现高并发请求处理的一个重要技术方案**

我把这个模型的特征用两个“三”来总结，也就是：



三类处理事件，即连接事件、写事件、读事件；



三个关键角色，即 reactor、acceptor、handler。



其实，Reactor 模型处理的是客户端和服务器端的交互过程，而这三类事件正好对应了客 户端和服务器端交互过程中，不同类请求在服务器端引发的待处理事件：

![image-20220814151212727](../../images/Redis/image-20220814151212727.png)





**三个关键角色**



首先，连接事件由 acceptor 来处理，负责接收连接；acceptor 在接收连接后，会创建 handler，用于网络连接上对后续读写事件的处理； 其次，读写事件由 handler 处理； 最后，在高并发场景中，连接事件、读写事件会同时发生，所以，我们需要有一个角色 专门监听和分配事件，这就是 reactor 角色。当有连接请求时，reactor 将产生的连接 事件交由 acceptor 处理；当有读写请求时，reactor 将读写事件交由 handler 处理。



![image-20220814151241138](../../images/Redis/image-20220814151241138.png)













# 单线程和epoll多路复用高性能

1）**基于内存；**

2）**单线程减少上下文切换，同时保证原子性；**

3）**IO多路复用；**

4）**高级数据结构（如 SDS、Hash以及跳表等）**。

**单线程＋IO多路复用**



**IO多路复用看以下博客**

必须要看

[张彦飞]: https://mp.weixin.qq.com/s/2y60cxUjaaE2pWSdCBX1lA



## 前提

**Redis是单线程吗？**

Redis 的单线程主要是指 Redis 的网络 IO 和键值对读写是由一个线程来完成的，这也是 Redis 对外提供键值存储服务的主要流程。但 Redis 的其他功能，比如持久化、异步删除、集群数据同步等，其实是由额外的线程执行的。



**Redis 单线程为什么还能这么快？**

因为它所有的数据都在**内存**中，所有的运算都是内存级别的运算，而且单线程避免了多线程的切换性能损耗问题。正因为 Redis 是单线程，所以要小心使用 Redis 指令，对于那些耗时的指令(比如keys)，一定要谨慎使用，一不小心就可能会导致 Redis 卡顿。 



**Redis 单线程如何处理那么多的并发客户端连接？**

Redis的**IO多路复用**：redis利用epoll来实现IO多路复用，将连接信息和事件放到队列中，依次放到文件事件分派器，事件分派器将事件分发给事件处理器。

![img](../../images/Redis/69701)









**自己看书的小总结（p161）：**多个客户端socket对应的fd经过epoll监控，哪个可读/可写了，放入文件队列，经文件时间分派器，分派给socket产生读写请求对应的文件事件。每一个网络连接其实都对应一个文件描述符（FD）。  I/O 多路复用模块同时监听多个 FD





## 原理与步骤



**前提**

epoll_wait调用时，仅仅观察这个list链表里有没有数据即可。有数据就返回，没有数据就sleep，等到timeout时间到后即使链表没数据也返回。所以，epoll_wait非常高效。



### **epoll_create** 

 **epoll_create** 用于创建一个 epoll 对象



### **epoll_ctl** 

**epoll_ctl** 用来给 epoll 对象添加或者删除一个 socket（将被监听的描述符添加到红黑树或从红黑树中删除或者对监听事件进行修改）





### **epoll_wait** 

**epoll_wait** 就是查看它当前管理的这些 socket 上有没有可读可写事件发生。（返回就绪事件的数量）

```c
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

- 阻塞等待注册的事件发生，返回事件的数目，并将触发的事件写入events数组中。
- events: 用来记录被触发的events，其大小应该和maxevents一致
- maxevents: 返回的events的最大个数
- 使用 epoll_wait 函数获取就绪的文件描述符





**如果所有描述符上都没有 IO 事件发生，该函数会阻塞，直到有事件到来。一旦有事件到来，epoll_wait 函数就返回。同时将所有发生的事件保存到events数组中，该数组的地址以及数组大小由你自己通过参数 events 和 maxevents 指定。同时 epoll_wait 会返回发生事件的个数**

如果 epoll_wait 返回了，会把所有发生的事件保存在数组 events 中





1. 当被监听的文件状态发生改变时（如socket接收到数据），会把文件句柄对应 `epitem` 对象添加到 `eventpoll` 对象的就绪队列 `rdllist` 中。并且把就绪队列的文件列表复制到 `epoll_wait()` 函数的 `events` 参数中。
2. 唤醒调用 `epoll_wait()` 函数被阻塞（睡眠）的进程



**引言**

> 在基于 epoll 的编程中，和传统的函数调用思路不同的是，我们并不能主动调用某个 API 来处理。因为无法知道我们想要处理的事件啥时候发生。所以只好提前把想要处理的事件的处理函数注册到一个**事件分发器**上去。当事件发生的时候，由这个事件分发器调用回调函数进行处理。这类基于实现注册事件分发器的开发模式也叫 Reactor 模型。
>
> redis的epoll回调函数就是上面的想要处理的事件的处理函数





**epoll知识图**

![image-20220812170429249](../../images/Redis/image-20220812170429249.png)

##  **回答（重点）**

先是**epoll_create** 创建一个epoll对象，**epoll_ctl()**把需要监控的socket加入到内核的红黑树里面。epoll采用事件驱动方式，当某个socket有事件发生时，就会通过回调函数将会加入到内核维护的就绪事件列表中。然后，redis封装的**epoll_wait**函数（aeApiPoll）循环操作就绪队列并且调用之前注册好的读/写回调函数，然后解析并查找命令并处理，添加到读/写对列里，将输出写到缓存等带发送。

（当用户调用 `epoll_wait()` 函数时，只会返回有事件发生的文件描述符的个数）









## Redis源码



关于redis处理客户端连接的源码

**主流程为下面两个函数**

```c

int main(int argc, char **argv) {
    ......
    // 启动初始化
    initServer();
    // 运行事件处理循环，一直到服务器关闭为止
    aeMain(server.el);
}
```



redis堆epoll进行了封装

```c
// 对应 epoll_create

static int aeApiCreate(aeEventLoop *eventLoop)

 

// 对应 epoll_ctl 添加事件

static int aeApiAddEvent(aeEventLoop *eventLoop, int fd, int mask)

// 对应 epoll_ctl 删除事件

static int aeApiDelEvent(aeEventLoop *eventLoop, int fd, int delmask)

 

// 对应 epoll_wait

static int aeApiPool(aeEventLoop *eventLoop, struct timeval *tvp)
```







###   initServer();

**在 initServer 这个函数内，Redis 做了这么三件重要的事情。**

- 创建一个 epoll 对象

- 绑定监听服务端口

-  **注册事件回调函数**  (注册一个 accept 事件处理器。accept 事件处理函数为 acceptTcpHandler)

  







###  aeMain(server.el);

在 aeMain 函数中，是一个无休止的循环，它是 Redis 中最重要的部分。在每一次的循环中，要做的事情可以总结为如下图。





![图片](../../images/Redis/640-16602964125267.png)



- 通过 epoll_wait 发现 listen socket 以及其它连接上的可读、可写事件
- 若发现 listen socket 上有新连接到达，则接收新连接，并追加到 epoll 中进行管理
- 若发现其它 socket 上有命令请求到达，则读取和处理命令，把命令结果写到缓存中，加入写任务队列
- 每一次进入 epoll_wait 前都调用 beforesleep 来将写任务队列中的数据实际进行发送
- 如若有首次未发送完毕的，当写事件发生时继续发送



```c
void aeMain(aeEventLoop *eventLoop) {

    eventLoop->stop = 0;
    while (!eventLoop->stop) {

        // 如果有需要在事件处理前执行的函数，那么运行它
        // 3.4 beforesleep 处理写任务队列并实际发送之
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);

        // 开始等待事件并处理
        // 3.1 epoll_wait 发现事件
        // 3.2 处理新连接请求
        // 3.3 处理客户连接上的可读事件
        aeProcessEvents(eventLoop, AE_ALL_EVENTS);
    }
}
```



#### epoll_wait 发现事件



Redis 不管有多少个用户连接，都是通过 **epoll_wait 来统一发现和管理其上的可读（包括 liisten socket 上的 accept事件）、可写事件的**。甚至连 timer，也都是交给 epoll_wait 来统一管理的。

![图片](../../images/Redis/640-16602969804769.png)





每当 epoll_wait 发现特定的事件发生的时候，就会调用相应的事先注册好的事件处理函数进行处理



```c
//file:src/ae.c
int aeProcessEvents(aeEventLoop *eventLoop, int flags)
{
    // 获取最近的时间事件
    tvp = xxx

    // 处理文件事件，阻塞时间由 tvp 决定
    numevents = aeApiPoll(eventLoop, tvp);
    for (j = 0; j < numevents; j++) {
        // 从已就绪数组中获取事件
        aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];

        //如果是读事件，并且有读回调函数
        fe->rfileProc()

        //如果是写事件，并且有写回调函数
        fe->wfileProc()
    }
}

//file: src/ae_epoll.c
static int aeApiPoll(aeEventLoop *eventLoop, struct timeval *tvp) {
    // 等待事件
    aeApiState *state = eventLoop->apidata;
    epoll_wait(state->epfd,state->events,eventLoop->setsize,
            tvp ? (tvp->tv_sec*1000 + tvp->tv_usec/1000) : -1);
    ...
}
```



**aeProcessEvents 就是调用 epoll_wait 来发现事件。当发现有某个 fd 上事件发生以后，则调为其事先注册的事件处理器函数 rfileProc 和 wfileProc。**















![img](https://mmbiz.qpic.cn/mmbiz_png/BBjAFF4hcwqfhvRDbibkvAZ10lKtSUKScmFaAGmwiaQsWSOoj4xZiamo3U6erdMF8YnBuCfP3YklLQnGBsdKQsasA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



























## Redis epoll总结图（良心总结）



![图片](../../images/Redis/640-16602958689625.png)



























# 持久化

Redis有两种持久化机制，RDB与AOF,下面是两种持久化机制的详解。

## RDB



**RDB持久化指在指定的时间间隔内通过内存快照将快照执行瞬间数据库的键值对数据以二进制的形式存放在RDB文件中（ dump.rdb ），也是默认的持久化方式。**





### RBD文件的载入

服务器在载入RDB文件期间，会一直处于阻塞状态，直到载入工作完成为止.



### 生成RDB文件



**SAVE:**

SAVE会阻塞Redis服务器进程，直到RDB文件创建完毕为止，在服务器进程阻塞期间，服务器不能处理任何命令请求



**BGSAVE:**

fork（）出一个子进程，然后由后台子进程负责创建RDB文件，服务器（父进程）继续处理命令请求



**两个对比**

| **命令**              | **save**         | **bgsave**                                     |
| --------------------- | ---------------- | ---------------------------------------------- |
| IO类型                | 同步             | 异步                                           |
| 是否阻塞redis其它命令 | 是               | 否(在生成子进程执行调用fork函数时会有短暂阻塞) |
| 复杂度                | O(n)             | O(n)                                           |
| 优点                  | 不会消耗额外内存 | 不阻塞客户端命令                               |
| 缺点                  | 阻塞客户端命令   | 需要fork子进程，消耗内存                       |



除了手动像上面的两种命令，Redis还支持自动保存



**自动间隔性保存**

Redis可以根据save选项设置的保存条件，**自动执行BGSAVE命令**。

RDB快照条件到了才会持久化

redisServer结构体有记录了保存条件的数组（struct saveparam *saveparams）,这个saveparams数组中的每一个saveparam都有两个属性（每个saveparam都是一个保存条件）

```c
struct saveparam {
 // 秒数
 time_t seconds; 
 //修改数
 int changes;
}
```

除了saveparams数组外，服务器状态还维持着一个dirty计数器，以及一个lastsave属性。

dirty:记录距离上一次成功执行SAVE命令或者BVGSAVE命令之后，服务器对数据库状态进行了多少次修改（包括写入、删除、更新等操作）



lastsave:记录上一次成功执行SAVE命令或者BGSAVE命令的时间。

```c
struct redisServer{
..............

//修改计数器

long long  dirty;

//上一次执行保存的时间

time_t lastsave;
................

}
```

根据上面四个属性，来检查保存条件是否满足来判断是否进行BGSAVE保存;

Redis的serverCron函数会遍历saveparams数组，通过dirty计数器和lastsave属性来和saveparam中的两个seconds和changes进行比较来判断是否进行RDB操作，只要saveparams数组所有保存条件的任意一个保存条件满足就会进行RDB。



### 优点;

- RDB 文件紧凑，**全量备份**，非常适合用于进行备份和灾难恢复
- 生成 RDB 文件时支持异步处理，主进程不需要进行任何磁盘IO操作
- RDB 在恢复大数据集时的速度比 AOF 的恢复速度要快

### 缺点：

RDB 快照是一次全量备份，存储的是内存数据的二进制序列化形式，存储上非常紧凑。**且在快照持久化期间修改的数据不会被保存**，可能丢失数据。



上面说到在快照持久化期间修改的数据不会被保存，好像有个**bgsave的写时复制(COW)机制**可以实现持久化期间修改的数据也可以保存。（下面引用仅作了解）

> Redis 借助操作系统提供的写时复制技术（Copy-On-Write, COW），在生成快照的同时，依然可以正常处理写命令。简单来说，bgsave 子进程是由主线程 fork 生成的，可以共享主线程的所有内存数据。bgsave 子进程运行后，开始读取主线程的内存数据，并把它们写入 RDB 文件。此时，如果主线程对这些数据也都是读操作，那么，主线程和 bgsave 子进程相互不影响。但是，如果主线程要修改一块数据，**那么，这块数据就会被复制一份，生成该数据的副本。然后，bgsave 子进程会把这个副本数据写入 RDB 文件，而在这个过程中，主线程仍然可以直接修改原来的数据**。



## AOF

全量备份总是耗时的，有时候我们提供一种更加高效的方式 AOF，其工作机制更加简单：会将每一个收到的写命令追加到文件中(这都是在主线程中执行)。



### 持久化步骤

**与RDB持久化通过保存数据库的键值对数据来记录数据库状态不同，AOF持久化是通过保存Redis服务器所执行的写命令来记录数据库状态的。**



**AOF持久化功能可以分为命令追加（写到aof缓冲区）、文件写入（写到内核AOF文件缓存区）、文件同步三个步骤**





AOF刷盘也是在主线程中

**写AOF文件是在主进程中执行，而AOF重写是在子进程中执行的。**



- **命令追加**

当AOF持久化功能处于打开状态，在执行完一个写命令之后，会以协议格式将被执行的写命令追加到服务器状态的aof_buf缓冲区的末尾

```c
struct redisServer{

//...

//AOF缓冲区
ads aof_buf;

//...
}
```



- **文件写入与同步**

  Redis的服务器进程就像一个事件循环（loop），这个循环中的文件时间负责接收客户端的命令请求，以及向客户端发生命令回复，而时间事件则负责执行像serverCron函数这样需要定时运行的函数。

​        因为服务器在处理文件事件时可能会执行写命令，使得一些内容被追加到aof_buf缓冲区里面，所以在服务器每次结束一个事件循环之前，它都会调用flushAppendOnlyFile函数，考虑是否需要将aof_buf缓冲区里面的内容写入和保存到AOF文件里面。



flushAppendOnlyFile函数的行为由服务器配置的appendfsync选项的值来决定。



| appendfsync值  | flushAppendOnlyFile函数行为                                  |
| -------------- | ------------------------------------------------------------ |
| always         | 将aof_buf所有内容写入并同步到AOF文件                         |
| everysec(默认) | 将aof_buf所有内容写到AOF文件，如果距离上层同步超过1秒，那么再次进行同步，这个同步是由另一个线程专门负责 |
| no             | 将aof_buf所有内容写到AOF文件，但并不对AOF文件进行同步，何时同步由操作系统决定。 |





**命令       -->   aof_buf缓冲区       -->     AOF文件**



**这个文件写入可能是写入内核文件缓冲区，同步才是真正写到磁盘上，将内核缓存区同步到磁盘上是由操作系统决定的。**





### AOF文件载入与还原

![img](../../images/Redis/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQxNDUzMjg1,size_16,color_FFFFFF,t_70.png)



### AOF重写

随着时间推移，AOF 持久化文件也会变的越来越大（命令一直追加，体积会变大）。为了解决此问题，Redis 提供了 `bgrewriteaof` 命令，作用是 fork 出一条新进程将内存中的数据以命令的方式保存到临时文件中，完成对**AOF 文件的重写**。



**注意：**

AOF文件重写并不需要对现有的文件进行任何读取、分析或者写入操作，而是依赖当前数据库状态，创建一个新的AOF文件，去除冗余数据，来减少旧AOF文件的体积过大问题。





**AOF重写是后台进行，放在子进程进行，这样做可以保证：**

1. 子进程在进行AOF重写期间，父进程可以继续处理命令请求。
2. 子进程带有服务器进程的数据副本，使用子进程而不是线程，可以在避免使用锁的情况下，保证数据的安全。





**数据不一致问题：**

在AOF重写期间，服务器还需要继续处理命令，而新的命令可能会对现有的数据库状态进行修改，从而使得服务器当前的数据库状态和重写后的AOF文件所保存的数据库状态不一致。



**解决:**

为了解决这种数据不一致问题，Redis服务器设置了一个AOF重写缓冲区，这个缓冲区在创建子进程之后开始使用（**也就是在AOF重写开始之后**），当Redis服务器执行完一个写命令之后，它会同时将这个写命令发送给**AOF缓冲区（aof_buf）**和**AOF重写缓冲区**。



**最后在AOF重写完成之后，子进程向父进程发送信号，将AOF重写缓冲区记录进行同步，这样就可以避免数据不一致问题。**



**注意：整个AOF过程，只有这个发送信号会对父进程进行阻塞，其他操作都不会进行阻塞。**









## 恢复数据

| **命令**   | **RDB**    | **AOF**      |
| ---------- | ---------- | ------------ |
| 启动优先级 | 低         | 高           |
| 体积       | 小         | 大           |
| 恢复速度   | 快         | 慢           |
| 数据安全性 | 容易丢数据 | 根据策略决定 |





### 用哪个好（总结）

RDB在子进程备份的时候，有新的数据是不会备份的，可能会丢数据，AOF不会丢数据

RDB全量备份太慢，AOF追加命令会快一点

RDB是子进程不影响住进程，AOF是在主线程执行的，在大量请求的时候，可能会影响效率。

AOF文件比RDB大







**优先使用AOF，如果AOF没有开那么使用RDB。**

官方推荐两个都启用。

如果只是做纯内存缓存，可以都不用。

如果对数据不敏感，可以选单独用RDB。

不建议单独用 AOF，因为可能会出现Bug。





> 如果你非常关心你的数据，但仍然可以承受数分钟以内的数据丢失，那么你可以只使用 RDB 持久。
>
> AOF 将 Redis 执行的每一条命令追加到磁盘中(这个追加是在主线程操作的)，处理巨大的写入会降低 Redis 的性能，不知道你是否可以接受。
>
> 数据库备份和灾难恢复：
>
> 定时生成 RDB 快照（snapshot）非常便于进行数据库备份， 并且 RDB 恢复数据集的速度也要比 AOF 恢复的速度要快。
>
> Redis 支持同时开启 RDB 和 AOF,系统重启后，Redis 会优先使用 AOF 来恢复数据，这样丢失的数据会最少。











### RDB优缺点



**优点：**

- RDB 文件紧凑，**全量备份**，非常适合用于进行备份和灾难恢复

- **生成 RDB 文件时支持异步处理，主进程不需要进行任何磁盘IO操作**

- **RDB 在恢复大数据集时的速度比 AOF 的恢复速度要快**



**缺点：**

RDB 快照是一次全量备份，存储的是内存数据的二进制序列化形式，存储上非常紧凑。**且在快照持久化期间修改的数据不会被保存**，可能丢失数据。全量备份总是耗时的。



上面说到在快照持久化期间修改的数据不会被保存，好像有个**bgsave的写时复制(COW)机制**可以实现持久化期间修改的数据也可以保存。





### AOF优缺点

**优点：**



AOF可以更好的保护数据不丢失。

> 1，使用 AOF 做持久化，可以设置不同的 fsync 策略，比如无 fsync ，每秒钟一次 fsync ，或者每次执行写入命令时 fsync 。
>
> AOF 的默认策略为每秒钟 fsync 一次，在这种配置下，Redis 仍然可以保持良好的性能，并且就算发生故障停机，也最多只会丢失一秒钟的数据。
>
> fsync 会在后台线程执行，所以主线程可以继续努力地处理命令请求。
>
> 2，AOF 文件是一个只进行追加操作的日志文件，不是生成新的之后替换掉那种，即使日志因为某些原因而包含了未写入完整的命令（比如写入时磁盘已满，写入中途停机，等等）， redis-check-aof 工具也可以轻易地修复这种问题。
>
> 3，Redis 可以在 AOF 文件体积变得过大时，自动地在后台对 AOF 进行重写： 重写后的新 AOF 文件包含了恢复当前数据集所需的最小命令集合。
>
> 整个重写操作是绝对安全的，因为 Redis 重写是创建新 AOF 文件，重写的过程中会继续将命令追加到现有旧的 AOF 文件里面，即使重写过程中发生停机，现有旧的 AOF 文件也不会丢失。 而一旦新 AOF 文件创建完毕，Redis 就会从旧 AOF 文件切换到新 AOF 文件，并开始对新 AOF 文件进行追加操作。



**缺点：**

对于相同的数据集来说，AOF 文件的体积通常要大于 RDB 文件的体积。恢复速度慢；

根据所使用的 fsync 策略，AOF 的速度可能会慢于 RDB。 







### 混合持久化

尽管 RDB 比 AOF 的数据恢复速度快，但是快照的频率不好把握：

- 如果频率太低，两次快照间一旦服务器发生宕机，就可能会比较多的数据丢失；
- 如果频率太高，频繁写入磁盘和创建子进程会带来额外的性能开销。

那有没有什么方法不仅有 RDB 恢复速度快的优点和，又有 AOF 丢失数据少的优点呢？

当然有，那就是将 RDB 和 AOF 合体使用，这个方法是在 Redis 4.0 提出的，该方法叫**混合使用 AOF 日志和内存快照**，也叫混合持久化。

如果想要开启混合持久化功能，可以在 Redis 配置文件将下面这个配置项设置成 yes：

```text
aof-use-rdb-preamble yes
```

混合持久化工作在 **AOF 日志重写过程**。

当开启了混合持久化时，在 AOF 重写日志时，`fork` 出来的重写子进程会先将与主线程共享的内存数据以 RDB 方式写入到 AOF 文件，然后主线程处理的操作命令会被记录在重写缓冲区里，重写缓冲区里的增量命令会以 AOF 方式写入到 AOF 文件，写入完成后通知主进程将新的含有 RDB 格式和 AOF 格式的 AOF 文件替换旧的的 AOF 文件。

也就是说，使用了混合持久化，AOF 文件的**前半部分是 RDB 格式的全量数据，后半部分是 AOF 格式的增量数据**。

![图片](../../images/Redis/f67379b60d151262753fec3b817b8617.png)

这样的好处在于，重启 Redis 加载数据的时候，由于前半部分是 RDB 内容，这样**加载的时候速度会很快**。

加载完 RDB 的内容后，才会加载后半部分的 AOF 内容，这里的内容是 Redis 后台子进程重写 AOF 期间，主线程处理的操作命令，可以使得**数据更少的丢失**。







 

# 主从与哨兵





## 主从同步策略

![img](../../images/Redis/80584)

主机master数据更新后根据配置和策略自动同步到从机slave，**Master**以**写**为主，**Slave**以**读**为主。



主从同步刚连接的时候进行**全量同步**，全量同步结束后开始增量同步。



如果有需要，slave 在任何时候都可以发起全量同步，其主要策略就是无论如何首先会尝试进行增量同步，如果失败则会要求 slave 进行全量同步，之后再进行增量同步。



**注意**：如果多个 slave 同时断线需要重启的时候，因为只要 slave 启动，就会和 master 建立连接发送SYNC请求和主机全量同步，如果多个同时发送 SYNC 请求，可能导致 master IO 突增而发送宕机。**所以我们要避免多个 slave 同时恢复重启的情况。**



**增量复制解释**

**增量复制**实际上就是在 slave 初始化完成后开始正常工作时 master 发生写操作同步到 slave 的过程。增量复制的过程主要是 master 每执行一个写命令就会向 slave 发送相同的写命令，slave 接受并执行写命令，从而保持主从一致。



## 主从复制原理

**全量**



![img](../../images/Redis/102424)

**原理步骤**

1. 如果你为master配置了一个slave，不管这个slave是否是第一次连接上Master，它都会发送一个**PSYNC**命令给master请求复制数据。

   

2. master收到PSYNC命令后，会在后台进行数据持久化通过bgsave生成最新的**rdb快照文件**，持久化期间，master会继续接收客户端的请求，它会把这些可能修改数据集的请求缓存在内存中。当持久化进行完毕以后，master会把这份rdb文件数据集发送给slave，**slave会把接收到的数据进行持久化生成rdb，然后再加载到内存中。然后，master再将之前缓存在内存中的命令发送给slave（在全量同步过程中，主机可能有写操作，所以得把同步之后主机的写操作放进缓存，达到数据一致性）。**

   

3. 当master与slave之间的连接由于某些原因而断开时，slave能够自动重连Master，如果master收到了多个slave并发连接请求，它只会进行一次持久化，而不是一个连接一次，然后再把这一份持久化的数据发送给多个并发连接的slave。



**增量复制**



Redis增量复制是指Slave初始化后开始正常工作时主服务器发生的写操作同步到从服务器的过程。 
增量复制的过程主要是主服务器每执行一个写命令就会向从服务器发送相同的写命令，从服务器接收并执行收到的写命令。





**断点续传**



![https://note.youdao.com/yws/public/resource/893c138fa39925f86b374fd46db322b4/xmlnote/D08B9F88D6AA4030A9C8A3C7FF0EEF37/102426](../../images/Redis/102426)





当master和slave断开重连后，一般都会对整份数据进行复制。但从redis2.8版本开始，redis改用可以支持部分数据复制的命令PSYNC去master同步数据，slave与master能够在网络连接断开重连后只进行部分数据复制(**断点续传**)。

master会在其内存中创建一个复制数据用的缓存队列，缓存最近一段时间的数据，master和它所有的slave都维护了复制的数据下标offset和master的进程id，因此，当网络连接断开后，slave会请求master继续进行未完成的复制，从所记录的数据下标开始。如果master进程id变化了，或者从节点数据下标offset太旧，已经不在master的缓存队列里了，那么将会进行一次全量数据的复制。







## **主从复制风暴**

如果有很多从节点，为了缓解**主从复制风暴**(多个从节点同时复制主节点导致主节点压力过大)，可以做如下架构，让部分从节点与从节点(与主节点同步)同步数据

![https://note.youdao.com/yws/public/resource/893c138fa39925f86b374fd46db322b4/xmlnote/2F8E815BDAE944A7ACBCD3B47F0C0D2B/102435](../../images/Redis/102435)





```
Q：从机可以进行写操作？
A：一般不可以。但可以通过配置文件让从服务器然后支持写操作。
```













## 哨兵架构



看书







  

# 集群



**主从与集群这块建议看书《Redis设计与实现》----- 挺好的书**



主从哨兵架构，主节点负责写，从结点负责读

集群只有主结点复制读写，从结点只是为了主结点挂了之后负责接替主结点工作，主结点可以水平扩容



## 脑裂

在哨兵架构中，redis的集群脑裂是某个master所在机器突然脱离了正常的网络，导致redis master节点跟redis slave节点和sentinel集群处于不同的网络分区，此时因为sentinel集群无法感知到master的存在，哨兵可能就会认为master宕机了，然后开启选举，将其他slave切换成了master，这个时候集群里就会有两个master，也就是所谓的脑裂。



出现集群脑裂后，如果客户端还在基于原来的master节点继续写入数据，那么新的master节点将无法同步这些数据，当网络问题解决之后，sentinel集群将原先的master节点降为slave节点，此时再从新的master中同步数据，将会造成大量的数据丢失。



master网络不通了，开始执行主从切换，主从切换还未完成，sentinel 不通知客户端用新的 master ，旧的master会接收命令工作。



**总结**

数据不一致只是结果，出现多个master才是脑裂问题，数据不一致是因为，两个网洛分区，分别像旧master和新master请求（旧master为成为新的master之前）

# 缓存



## 缓存雪崩



**同一时刻大量key失效，请求之间打到数据库上，数据库肯定支撑不住挂了。**

**同一时间大面积失效，那一瞬间Redis跟没有一样，那这个数量级别的请求直接打到数据库几乎是灾难性的，你想想如果打挂的是一个用户服务的库，那其他依赖他的库所有的接口几乎都会报错，如果没做熔断等策略基本上就是瞬间挂一片的节奏，你怎么重启用户都会把你打挂**



### 应对

**一句话：对key的过期时间设置随机值**



处理缓存雪崩简单，在批量往**Redis**存数据的时候，把每个Key的失效时间都加个随机值就好了，这样可以保证数据不会在同一时间大面积失效，我相信，Redis这点流量还是顶得住的。

如果**Redis**是集群部署，将热点数据均匀分布在不同的**Redis**库中也能避免全部失效的问题，不过本渣我在生产环境中操作集群的时候，单个服务都是对应的单个**Redis**分片，是为了方便数据的管理，但是也同样有了可能会失效这样的弊端，失效时间随机是个好策略。

或者设置热点数据永远不过期，有更新操作就更新缓存就好了（比如运维更新了首页商品，那你刷下缓存就完事了，不要设置过期时间），电商首页的数据也可以用这个操作，保险。















## 缓存击穿

**缓存击穿**，这个跟**缓存雪崩**有点像，但是又有一点不一样，**缓存雪崩**是因为大面积的缓存失效，打崩了DB，而缓存击穿不同的是**缓存击穿**是指一个Key非常热点，在不停的扛着大并发，大并发集中对这一个点进行访问，当这个Key在失效的瞬间，持续的大并发就穿破缓存，直接请求数据库，就像在一个完好无损的桶上凿开了一个洞。





热点key突然过期，所有请求打到db上，db扛不住



### 解决

**一句话，缓存击穿的话，设置热点数据永远不过期。或者加上互斥锁就能搞定了**



1. 提高缓存高可用；
2. 保证热点数据存在缓存；
3. 访问慢设备时由高并发转低并发（一般使用分布式锁）；













## 缓存穿透

缓存穿透是指缓存和数据库中都没有的数据，而用户不断发起请求，我们数据库的 id 都是1开始自增上去的，如发起为id值为 -1 的数据或 id 为特别大不存在的数据。这时的用户很可能是攻击者，攻击会导致数据库压力过大，严重会击垮数据库。



### 解决

1. **接口校验**
2. **缓存对应key的value为null**
3. **布隆过滤器**



**缓存穿透**我会在接口层增加校验，比如用户鉴权校验，参数做校验，不合法的参数直接代码Return，比如：id 做基础校验，id <=0的直接拦截等。

**从缓存取不到的数据，在数据库中也没有取到，这时也可以将对应Key的Value对写为null、位置错误、稍后重试这样的值具体取啥问产品，或者看具体的场景，缓存有效时间可以设置短点，如30秒（设置太长会导致正常情况也没法使用）。**

这样可以防止攻击用户反复用同一个id暴力攻击，但是我们要知道正常用户是不会在单秒内发起这么多次请求的，那网关层**Nginx**本渣我也记得有配置项，可以让运维大大对单个IP每秒访问次数超出阈值的IP都拉黑。



**别的办法**



### 布隆过滤器



还有我记得**Redis**还有一个高级用法**布隆过滤器（Bloom Filter）**这个也能很好的防止缓存穿透的发生，他的原理也很简单就是利用高效的数据结构和算法快速判断出你这个Key是否在数据库中存在，不存在你return就好了，存在你就去查了DB刷新KV再return。

![https://note.youdao.com/yws/public/resource/7b6df00d88f1554d79b2d688c23148a2/xmlnote/CB877F64DE984480871C578364D570B4/81509](../../images/Redis/81509)

**作用：**对于恶意攻击，向服务器请求大量不存在的数据造成的缓存穿透，还可以用布隆过滤器先做一次过滤，对于不存在的数据布隆过滤器一般都能够过滤掉，不让请求再往后端发送。当布隆过滤器说**某个值存在时，这个值可能不存在；当它说不存在时，那就肯定不存在。**

   





**原理**：**布隆过滤器由一个初值都为 0 的 bit 数组和 N 个哈希函数组成，可以用来快速判断某个数据是否存在。**

- 首先，使用 N 个哈希函数，分别计算这个数据的哈希值，得到 N 个哈希值。
- 然后，我们把这 N 个哈希值对 bit 数组的长度取模，得到每个哈希值在数组中的对应位置。
- 最后，我们把对应位置的 bit 位设置为 1，这就完成了在布隆过滤器中标记数据的操作。

- 如果数据不存在（例如，数据库里没有写入数据），我们也就没有用布隆过滤器标记过数据，那么，bit 数组对应 bit 位的值仍然为 0。

**总结：**当查询某个数据时，先得到这个数据在bit数组对应的N个位置，看看这几个位置上的bit值是否是1，只要有一个不为1，那么数据库一定没有这个值



### 布隆过滤器缺陷

1. 它在判断元素是否在集合中时是有一定错误几率的，比如它会把不是集合中的元素判断为处在集合中；

2. 不支持删除元素。



关于第一个缺陷，主要是 Hash 算法的问题。**因为布隆过滤器是由一个二进制数组和一个 Hash 算法组成的，Hash 算法存在着一定的碰撞几率。Hash 碰撞的含义是不同的输入值经过 Hash 运算后得到了相同的 Hash 结果。



**解决：使用多个 Hash 算法为元素计算出多个 Hash 值，只有所有 Hash 值对应的数组中的值都为 1 时，才会认为这个元素在集合中。**





### 布隆过滤器与bitmap对比





bitmap多用于海里整数判断是否存在某个数

如果数范围很大，但是数量很少，比如10范围1~1亿级别的数字，即使bitmap相比于其他数据结构已经很省空间了，但是还有更省空间的

这时候布隆过滤器登场了

**它也是基于位数组，采用多个哈希函数来计算key值并在位数组相应的下标赋1，这样10个数字加上 对应的n个哈希函数所占的位很少，构建布隆过滤器开辟的空间很小，布隆过滤器说不存在，绝对不存在。**





[布隆过滤器与bitmap]: https://cloud.tencent.com/developer/article/1813682







## 本地缓存与redis缓存

本地缓存：jvm里面的缓存











#  数据库和缓存如何保证一致性？



**在大并发下，同时操作数据库与缓存会存在数据不一致性问题**

















### 先言总结

直接看这个

https://mp.weixin.qq.com/s/HXGtd6YRtpn6oH5GvK-B5Q

我直接先抛一下结论：**在满足实时性的条件下，不存在两者完全保存一致的方案，只有最终一致性方案。** 根据网上的众多解决方案，总结出 6 种，直接看目录：



![图片](../../images/Redis/640-166031000390815.png)







#### **方案比较**

![image-20220605201510623](../../images/Redis/image-20220605201510623.png)





#### **最终结论**



![image-20220605201518586](../../images/Redis/image-20220605201518586.png)













## 延时双删（先删Redis再写mysql再删Redis）



对于“先删除 Redis，再写 MySQL”，如果要解决最后的不一致问题，其实再对 Redis 重新删除即可，**这个也是大家常说的“延时双删”。**

![图片](../../images/Redis/640-166031024772617.png)





为了便于大家看图，对于蓝色的文字，“删除缓存 10”必须在“回写缓存10”后面，那如何才能保证一定是在后面呢？**网上给出的第一个方案是，让请求 A 的最后一次删除，等待 500ms。**



对于这种方案，看看就行，反正我是不会用，太 Low 了，风险也不可控。

**那有没有更好的方案呢，我建议异步串行化删除，即删除请求入队列**



![图片](../../images/Redis/640-166031035006819.png)





异步删除对线上业务无影响，串行化处理保障并发情况下正确删除。

如果双删失败怎么办，网上有给 Redis 加一个缓存过期时间的方案，这个不敢苟同。**个人建议整个重试机制，可以借助消息队列的重试机制，也可以自己整个表，记录重试次数**，方法很多。



**简单小结一下：**

- “缓存双删”不要用无脑的 sleep 500 ms；
- 通过消息队列的异步&串行，实现最后一次缓存删除；
- 缓存删除失败，增加重试机制。













## 先写mysql再删除redis

![图片](../../images/Redis/640-166031086270221.png)





对于上面这种情况，对于第一次查询，请求 B 查询的数据是 10，但是 MySQL 的数据是 11，**只存在这一次不一致的情况，对于不是强一致性要求的业务，可以容忍。**（那什么情况下不能容忍呢，比如秒杀业务、库存服务等。）

当请求 B 进行第二次查询时，因为没有命中 Redis，会重新查一次 DB，然后再回写到 Reids。

















## 先写Mysql再通过binlog异步更新redis



这种方案，主要是监听 MySQL 的 Binlog，然后通过异步的方式，将数据更新到 Redis，这种方案有个前提，查询的请求，不会回写 Redis。

![图片](../../images/Redis/640-166031090672723.png)



这个方案，会保证 MySQL 和 Redis 的最终一致性，但是如果中途请求 B 需要查询数据，如果缓存无数据，就直接查 DB；如果缓存有数据，查询的数据也会存在不一致的情况。

**所以这个方案，是实现最终一致性的终极解决方案，但是不能保证实时性。**















# 原子操作







## 并发问题

![img](../../images/Redis/dce821cd00c1937b4aab1f130424335c.jpg)





正确答案redis应该是8，但此时是9，因为并发修改产生了数据更新错误。









**Redis不是单线程吗？怎么还会有并发读写的问题呢**？



这里的并发是指有多个客户端同时访问Redis，而客户端执行的业务逻辑不只一个命令操作。假设客户端A要先从Redis读数据，然后做了修改把数据再写回Redis，此时，在读数据和写回数据的期间就可能有其他客户端的并发操作执行，所以仍然存在并发读写问题。





为了保证并发访问的正确性，Redis 提供了两种方法，分别是**加锁**和原子操作。





## 加锁



把上面的三个读取数据、修改数据、写回数据并行操作变为串行操作



但加锁会使系统并发性能降低





## 原子操作



Redis 的原子操作采用了两种方法：



- **1.把多个操作在 Redis 中实现成一个操作，也就是单命令操作；**

  

Redis 提供了 INCR/DECR 命令，把这三个操作转变为一个原子操作了。INCR/DECR 命令可以对数据进行增值 / 减值操作，



- **2.把多个操作写到一个 Lua 脚本中，以原子性方式执行单个 Lua 脚本。**



果我们要执行的操作不是简单地增减数据，而是有更加复杂的判断逻辑或者是其他操作，那么，Redis 的单命令操作已经无法保证多个操作的互斥执行了。所以，这个时候，我们需要使用第二个方法，也就是 Lua 脚本。









# 事务

Redis通过MULTI、EXEX、WATCH等命令来实现事务功能。

事务提供了一种将多个目录请求打包，然后一次性、按顺序地执行多个命令的机制，并且在事务执行期间，服务器不会中断事务而区执行其他客户端的命令请求，它会将事务中的命令都执行完毕，然后才会处理其他客户端的命令请求。





- MULTI：开启事务
- EXEX：提交事务
- WATCH：监视键



事务的三个阶段：

- 事务开始
- 命令入队
- 事务执行



**Redis 中并没有提供回滚机制。虽然 Redis 提供了 DISCARD 命令，但是，这个命令只能用来主动放弃事务执行，把暂存的命令队列清空，起不到回滚的效果。**











## WATCH

WATCH命令是一个乐观锁，它可以在EXEC命令执行之前，监视任意数量的数据库键，并在EXEC命令执行时，检查被监视的键是否至少有一个已经被修改过了，如果是的话，服务器将拒绝执行事务，并且向客户端返回代表事务执行失败的空回复。



每个Redis数据库都保持一个**wactch_keys**字典，键是某个被WATCH命令监视的数据库键，而字典的值则是一个链表，链表记录了所有监视相应数据库键的客户端。





**监视机制的触发**

所有对数据库进行修改的命令SET,LPUSH等等，在执行之后都会调用multi.c/touchWatchKey函数对watch_keys字典进行检查，查看是否有客户端正在监视刚刚被命令修改过的数据库键。如果有的话，那么touchWatchKey函数会将监视被修改的键的客户端的REDIS_DIRTY_CAS标识打开，**表示该客户端的事务安全性已经被破坏。**









## 判断事务是否安全

![image-20220512133249672](../../images/Redis/image-20220512133249672.png)







## ACID

### 原子性（得分类讨论）

mysql事务原子性：数据库将事务的多个操作当作一个整体执行，要么都执行，要么一个也不执行

相比于mysql,Redis 事务执行过程中，如果一个命令执行出错，那么就返回错误，然后还是会接着继续执行下面的命令。





**redis原子性得分情况讨论**





**原子性总结：**



**命令入队时候就报错，会放弃事务执行，毕竟报错前的命令还在队列里面没有执行，保证了原子性。**

**命令入队时没保错，实际执行时保错，但此时redis只会对错误命令报错，其他正确命令还会执行。这个时候没有保证原子性。**





![image-20220610173724594](../../images/Redis/image-20220610173724594.png)

### 一致性



再说一次，Redis不支持事务回滚

不论事务是否执行成功，数据库执行事务前后要一致。

没有约束，满足一致性。



事务里面，redis只会执行正确的命令，可以保证一致性。

事务的一致性保证会受到错误命令、实例故障的影响。所以，我们按照命令出错和实例故障的发生时机，分成三种情况来看。

- 情况一：命令入队时就报错在这种情况下，事务本身就会被放弃执行，所以可以保证数据库的一致性。
- 情况二：命令入队时没报错，实际执行时报错在这种情况下，有错误的命令不会被执行，正确的命令可以正常执行，也不会改变数据库的一致性
- 情况三：EXEC 命令执行时实例发生故障



### 隔离性

即使有多个事务并发执行，各个事务之间不影响

因为Redis使用单线程的方式来执行事务，并且服务器保证，在执行事务期间不会对事务进行中断，因此，Redis的事务总是以串行的方式运行，并且事务也总是具有隔离性的。





### 耐久性



当一个事务执行完毕后，执行这个事务的结果已经被保存到硬盘里面了，即使事务执行后停机，执行事务所的的结果也不会丢失。





# 分布式锁



https://time.geekbang.org/column/article/301092



## Redis分布式锁

加锁, unique_value作为客户端唯一性的标识

### 分布式锁应用场景

**集群部署**

系统A和系统B是两个部署在不同节点的相同应用（集群部署），这时客户端请求传来，两个系统都受到了请求，并且该请求是对数据表进行插入操作，如果这个时候不加锁来控制，可能会导致数据库新增两条记录，这时系统也不能允许的，由于是在不同应用内，在单个应用内加JVM级别的锁，另一个应用是感知不到的，这时需要用到分布式锁。



**分布式部署**

多个结点部署不同的业务



比如：一个订单，客户正在前台修改地址，管理员在后台同时修改备注。地址和备注字段的修改，都必须正确更新，这二个请求同时到达的话，如果不借助db的事务，很容易造成行锁竞争，但用事务的话，db的性能显然比不上redis轻量。

解决思路：A,B二个请求，谁先抢到分布式锁（假设A先抢到锁），谁先处理，抢不到的那个（即：B），在一旁不停等待重试，重试期间一旦发现获取锁成功，即表示A已经处理完，把锁释放了。这时B就可以继续处理了。










### 加锁lock

`Redis`的`setnx`命令是当`key`不存在时设置`key`，**但`setnx`不能同时完成`expire`设置失效时长，不能保证`setnx`和`expire`的[原子性](https://so.csdn.net/so/search?q=原子性&spm=1001.2101.3001.7020)。**我们可以使用`set`命令完成`setnx`和`expire`的操作，并且这种操作是[原子操作](https://so.csdn.net/so/search?q=原子操作&spm=1001.2101.3001.7020)。（如果用setnx命令，设置时长用expire不能保证原子性，所以用set命令）

**set 命令 两个参数**

 NX:key不存在时设置value，成功返回OK，失败返回(nil)

PX:PX milliseconds：设置失效时长，单位毫秒 

```redis
SET lock_key unique_value NX PX 10000 ( PX 10000指这个key10000秒后过期)
```





**SET命令加上nx参数使用，如果redis中没有这个key，则设置这个键值对（设置成功则表示加锁成功），如果有则，则不做操作（说明其他客户端可能加锁成功了**



**即使多个同时请求加锁。因为 Redis 使用单线程处理请求，所以，即使客户端 A 和 C 同时把加锁请求发给了 Redis，Redis 也会串行处理它们的请求。**





```java
public boolean lock(String id) {
    Long start = System.currentTimeMillis();
    try {
        for (; ; ) {
            //SET命令返回OK ，则证明获取锁成功
            String lock = jedis.set(LOCK_KEY, id, params);
            if ("OK".equals(lock)) {
                return true;
            }
            //否则循环等待，在timeout时间内仍未获取到锁，则获取失败
            long l = System.currentTimeMillis() - start;
            if (l >= timeout) {
                return false;
            }
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    } finally {
        jedis.close();
    }
}







```



### 释放锁unlock



```redis
DEL lock_key
```

就是删除这个键值对



**解锁就是删除key**

```java
public boolean unlock(String id) {
    String script =
            "if redis.call('get',KEYS[1]) == ARGV[1] then" +
                    "   return redis.call('del',KEYS[1]) " +
                    "else" +
                    "   return 0 " +
                    "end";
    try {
        String result = jedis.eval(script, Collections.singletonList(LOCK_KEY), Collections.singletonList(id)).toString();
        return "1".equals(result) ? true : false;
    } finally {
        jedis.close();
    }
}
```



### 风险以及改进



#### ①

第一个风险是，假如某个客户端在执行了 SETNX 命令、加锁之后，紧接着却在操作共享数据时发生了异常，结果一直没有执行最后的 DEL 命令释放锁。因此，锁就一直被这个客户端持有，其它客户端无法拿到锁，也无法访问共享数据和执行后续操作，这会给业务应用带来影响。



**解决**

**给锁变量设置一个过期时间**

#### ②

**保证谁加锁，谁解锁，不能我加的锁，别人给删了。**

我们再来看第二个风险。如果客户端 A 执行了 SETNX 命令加锁后，假设客户端 B 执行了 DEL 命令释放锁，此时，客户端 A 的锁就被误释放了。如果客户端 C 正好也在申请加锁，就可以成功获得锁，进而开始操作共享数据。这样一来，客户端 A 和 C 同时在对共享数据进行操作，数据就会被修改错误，这也是业务层不能接受的。



**解决**

区分来自不同客户端的锁操作，可以让每个客户端给锁变量设置一个唯一值UUID，这里的唯一值就可以用来标识当前操作的客户端。在释放锁操作时，客户端需要判断，当前锁变量的值是否和自己的唯一标识相等，只有在相等的情况下，才能释放锁。这样一来，就不会出现误释放锁的问题了。





### 缺点

而且，锁一般都是需要可重入行的，上面的线程都是执行完了就释放了，无法再次进入了，进去也是重新加锁了，对于一个锁的设计来说肯定不是很合理的。

我不打算手写，因为都有现成的，别人帮我们写好了。



下面的**redisson**就有锁重入的原理





## Redlock



**上面讲的是单个Redis结点的分布式锁**



**Redlock是基于多个 Redis 节点实现高可靠的分布式锁**



Redlock 算法的基本思路，是让客户端和多个独立的 Redis 实例依次请求加锁，如果客户端能够和半数以上的实例成功地完成加锁操作，那么我们就认为，客户端成功地获得分布式锁了，否则加锁失败。这样一来，即使有单个 Redis 实例发生故障，因为锁变量在其它实例上也有保存，所以，客户端仍然可以正常地进行锁操作，锁变量并不会丢失。



**步骤：**

1. 第一步是，客户端获取当前时间。

   

2. 第二步是，客户端按顺序依次向 N 个 Redis 实例执行加锁操作。这里的加锁操作和在单实例上执行的加锁操作一样，使用 SET 命令，带上 NX，EX/PX 选项，以及带上客户端的唯一标识。当然，如果某个 Redis 实例发生故障了，为了保证在这种情况下，Redlock 算法能够继续运行，我们需要给加锁操作设置一个超时时间。如果客户端在和一个 Redis 实例请求加锁时，一直到超时都没有成功，那么此时，客户端会和下一个 Redis 实例继续请求加锁。加锁操作的超时时间需要远远地小于锁的有效时间，一般也就是设置为几十毫秒。

   

3. 第三步是，一旦客户端完成了和所有 Redis 实例的加锁操作，客户端就要计算整个加锁过程的总耗时。客户端只有在满足下面的这两个条件时，才能认为是加锁成功。条件一：客户端从超过半数（大于等于 N/2+1）的 Redis 实例上成功获取到了锁；条件二：客户端获取锁的总耗时没有超过锁的有效时间。



**在满足了这两个条件后，我们需要重新计算这把锁的有效时间，计算的结果是锁的最初有效时间减去客户端为获取锁的总耗时。如果锁的有效时间已经来不及完成共享数据的操作了，我们可以释放锁，以免出现还没完成数据操作，锁就过期了的情况。**





## redisson



Redisson是**java的redis客户端之一**，提供了一些api方便操作redis



**实现了可重入**



Redisson普通的锁实现源码主要是**RedissonLock**这个类。



源码中**加锁/释放锁**操作都是用**lua**脚本完成的，封装的非常完善，开箱即用



**源码暂时还没看**





# 跳表

![img](../../images/Redis/webp.webp)



**参考**

https://www.jianshu.com/p/9d8296562806

跳表的实现代码，skiplist；



```java
class Skiplist {
    /*
    Redis底层数据结构:跳表->以O(logN)的时间复杂度内完成增删改查指定元素
    核心原理:建立多层随机索引,将时间复杂度降至logN,具体可参考简书的博客
    重要函数:1.findClosest(node,level,num) (从curNode开始)寻找level内最靠近target的左边节点
            该位置就是在该层内最接近target且<target的位置,将来用于完成插入、删除、查找等操作
            2.randomLevel() 以0.25为概率因子生成最大值为32的随机数
            数字每增加1,出现的概率为前一个数的0.25倍
            如:1出现的概率为0.75;1以上出现的概率为0.25;
            2出现的概率就是0.25*0.75;3出现的概率就是0.25*0.75*0.75
            .......这样可以间接保证每一层的元素成类似于二叉树的结构,最大高度不过logN
    */
    // 随机数概率因子
    private static final double P_FACTOR = 0.25;
    // 最大层数
    private static final int MAX_LEVEL = 32;
    // 当前实际有效的最大层数
    private int curLevel;
    // 头结点
    Node head;

    /*
    节点类:值+同层右边的节点引用数组
    */
    class Node {
        int val;
        Node[] next;//每个结点，每一层对应的下一个结点，以数组存储
        
        public Node(int _val, int level) {
            val = _val;
            next = new Node[level];
        }
    }

    /*
    用给定规则生成随机数(非静态方法也可以调用静态变量)
    */
    private int randomLevel() {
        // 层数level从1开始
        int level = 1;
        while(Math.random() < P_FACTOR && level < MAX_LEVEL) {
            level++;
        }
        return level;
    }

    /*
    跳表构造器
    */
    public Skiplist() {
        // 一开始啥也没
        curLevel = 0;
        // head指出32个next的节点
        head = new Node(-1, MAX_LEVEL);
    }
    
    /*
    从curNode开始:
    寻找level层内最靠近target的左边节点(如target=6,5->6->7,停留在5;如target=6,5->7->8,停留在5)
    */
    private Node findClosest(Node curNode, int level, int target) {
        // curNode就是上一层下沉位置的开端
        while(curNode.next[level] != null && curNode.next[level].val < target) {
            curNode = curNode.next[level];
        }
        return curNode;
    }

    /*
    查找跳表内是否存在target元素
    */
    public boolean search(int target) {
        Node curNode = head;
        // 从有效的顶层开始搜索
        for(int i = curLevel - 1; i >= 0; i--) {
            // searchNode停留在<=target的左边
            curNode = findClosest(curNode, i, target);
            if(curNode.next[i] != null && curNode.next[i].val == target) {
                // 找到一个即可返回true
                return true;
            }
        }
        // 找遍所有层都找不到就返回false
        return false;
    }
    
    /*
    往跳表内添加元素num
    */
    public void add(int num) {
        // 随机层
        int level = randomLevel();
        // 随机层对应的新节点
        Node newNode = new Node(num, level);
        Node curNode = head;
        for(int i = curLevel - 1; i >= 0; i--) {
            // 找到要插入的位置
                //在每层找到最靠近target的左结点
            curNode = findClosest(curNode, i, num);
            // 当前层<newNode要求的层数时,每一层均要进行插入操作
            // 如level=2时,只要求索引为0,1的层进行插入
            if(i < level) {
                if(curNode.next[i] == null) {
                    // 目标位置处为空直接接上
                    curNode.next[i] = newNode;
                }else {
                    // 否则就要进行标准插入操作
                    Node tmp = curNode.next[i];
                    curNode.next[i] = newNode;
                    newNode.next[i] = tmp;
                }
            }
        }
        // 此时有可能要更新curLevel
          //此时有可能要更新curLevel,因为有可能随机层大于最大有效层,那就单打把大于有效层的索引加入
        if(level > curLevel) {
            for(int i = curLevel; i < level; i++) {
                head.next[i] = newNode;
            }
            curLevel = level;
        }
    }
    
    /*
    往跳表内删除元素num
    */
    public boolean derase(int num) {
        Node curNode = head;
        // 删除成功标记
        boolean flag = false;
        for(int i = curLevel - 1; i >= 0; i--) {
            curNode = findClosest(curNode, i, num);
            // 可能有多个相同的值(如果存在多个num删除其中任意一个即可)
            // 当出现多个重复值且前面节点层数低后面层数高,一层一层删下来其实删除的不是同一个节点
            // 但是不会影响最终结果
            if(curNode.next[i] != null && curNode.next[i].val == num) {
                // 进行删除操作
                curNode.next[i] = curNode.next[i].next[i];
                // 标记删除成功
                flag = true;
            }
        }
        return flag;
    }
}
```

**插入最难理解**





**步骤**

1. 先随机出来一个层数，new要插入的节点，先叫做插入节点newNode；

   

2. 根据跳表实际的总层数从上往下分析，要插入一个节点newNode时，先找到节点在该层的位置：因为是链表，所以需要一个节点node，满足插入插入节点newNode的值刚好不大于node的下一个节点值，当然，如果node的下个节点为空，那么也是满足的。

   

3. 我们先把找节点在某一层位置的方法封装起来：findClosest函数

   

4. 
   确定插入节点newNode在该层的位置后，先判断下newNode的随机层数是否小于当前跳表的总层数，如果是，则用链表的插入方法将newNode插入即可。

   

5. 如此循环，直到最底层插入newNode完毕。

   

6. 循环完毕后，还需要判断下newNode随机出来的层数是否比跳表的实际层数还要大，如果是，直接将超过实际层数的跳表的头节点指向newNode即可，该跳表的实际层数也就变为newNode的随机层数了。





## 并发安全版本

用CAS来实现并发安全，例如findClosest找到在每层找到最靠近target的左结点后（设这个结点叫node），CAS的替换这个node.next,依次来达到并发安全。





# 延时队列



## 场景

1.对于红包场景，账户 A 对账户 B 发出红包通常在 1 天后会自动归还到原账户。

2.对于实时支付场景，如果账户 A 对商户 S 付款 100 元，5秒后没有收到支付方回调将自动取消订单。

3.订单未支付到时间自动取消订单

## 解决方案



### 方案一：

采用通过定时任务采用数据库/非关系型数据库轮询方案。

优点：

- 实现简单，对于项目前期这样是最容易的解决方案。

缺点：

1. DB 有效使用率低，需要将一部分的数据库的QPS分配给 JOB 的无效轮询。
2. 服务资源浪费，因为轮询需要对所有的数据做一次 SCAN 扫描 JOB 服务的资源开销很大。

### 方案二：

采用延迟队列：

优点：

1.  服务的资源使用率较高，能够精确的实现超时任务的执行。
2. 减少 DB 的查询次数，能够降低数据库的压力

缺点：

- 对于延迟队列来说本身设计比较复杂，目前没有通用的比较好过的方案。





**下面为redis延时队列实现**



先来看两个redis命令

 Zrangebyscore命令：返回有序集 key 中，所有 score 值介于 min 和 max 之间(包括等于 min 或 max )的成员





```Redis
//返回所有符合条件 1 < score <= 5 的成员
ZRANGEBYSCORE zset (1 5

//返回所有符合条件 1 < score <= 5 的成员
ZRANGEBYSCORE zset (5 (10
```



### 方案三

用消息[中间件 ](https://cloud.tencent.com/product/tdmq?from=10680)**rabbitmq**实现延时队列



## 方案二思路

把当前时间戳和延时时间相加，也就是到期时间，存入Redis中，然后不断轮询，找到到期的，拿到再删除即可。



![img](../../images/Redis/509087-20200404111242306-943787231.png)











使用RedisTemplate作为Redis客户端连接,放在延时队列中的可以看成是一个个延时任务。

**socre**为当前时间+延时时间

**value**是任务(json string格式)

**Redis zrem** ， 命令用于**移除有序集中的一个或多个成员，不存在的成员将被忽略**。

**rangeByScore**：返回zset结构中，score值位于min到max的成员。





```java
@Data
public class DelayTask<T> {
	// 消息id
    private String id;
    // 任务名称
    private String taskName;
    // 具体任务内容
    private T msg;

}

```



**延时队列具体实现，主要分为放任务和轮询任务两个部分**



## 1放任务+2轮询任务（具体步骤）

```java
public class RedisDelayQueue<T> {
    /**
     * 延迟队列名称
     */
    private String delayQueueName = "delayQueue";
  
    // 传入redis客户端操作
    public RedisDelayQueue(RedisTemplate redisTemplate,String delayQueueName)
    {
        this.redisTemplate = redisTemplate;
        this.delayQueueName = delayQueueName;
    }
    /**
     * 设置延迟消息
     */
    public void setDelayTasks(T msg, long delayTime) {
        DelayTask<T> delayTask = new DelayTask<>();
        delayTask.setId(UUID.randomUUID().toString());
        delayTask.setMsg(msg);
        
        //socre为当前时间+延时时间
        Boolean addResult = redisTemplate.opsForZSet().add(delayQueueName（key）, JSONObject.toJSONString(delayTask)(value), System.currentTimeMillis() + delayTime) (score);
        if(addResult)
        {
            System.out.println("添加任务成功！"+JSONObject.toJSONString(delayTask)+"当前时间为"+ LocalDateTime.now());
        }
    }
    /**
     * 监听延迟消息
     */
    public void listenDelayLoop() {
        while (true) {
            // 获取一个到点的消息
            
            //拿比当前时间前一点的一条数据进行消费，查询最早的一条任务，来进行消费(返回的是范围内第一条数据，从小到大排列)
            //从0到当前时间，如果有返回，那么返回的肯定是过期的
            Set<String> set = redisTemplate.opsForZSet().rangeByScore(delayQueueName （key）, 0 (min), System.currentTimeMillis() (max), 0(offset), 1 (count));

            // 如果没有，就等等
            if (set.isEmpty()) {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                // 继续执行
                continue;
            }
            // 获取具体消息的key
            String it = set.iterator().next();
            // 删除成功，zrem命令删除zset里面的value
            if (redisTemplate.opsForZSet().remove(delayQueueName, it) > 0) {
                // 拿到任务
                DelayTask delayTask = JSONObject.parseObject(it, DelayTask.class);
                // 后续处理
                System.out.println("消息到期"+delayTask.getMsg().toString()+",时间为"+LocalDateTime.now());
            }
        }
    }
}

```







## lua脚本优化

上面的算法中同一个任务可能会被多个进程取到之后再使用 zrem 进行争抢，那些没抢到 的进程都是白取了一次任务，这是浪费。可以考虑使用 lua scripting 来优化一下这个逻辑，将 zrangebyscore 和 zrem 一同挪到服务器端进行原子化操作，这样多个进程之间争抢任务时就不 会出现这种浪费了









# Redis内存淘汰策略



Redis 内存淘汰策略共有八种，这八种策略大体分为「不进行数据淘汰」和「进行数据淘汰」两类策略。

***1、不进行数据淘汰的策略***

**noeviction**（Redis3.0之后，默认的内存淘汰策略） ：它表示当运行内存超过最大设置内存时，不淘汰任何数据，而是不再提供服务，直接返回错误。

***2、进行数据淘汰的策略***

针对「进行数据淘汰」这一类策略，又可以细分为「在设置了过期时间的数据中进行淘汰」和「在所有数据范围内进行淘汰」这两类策略。 在设置了过期时间的数据中进行淘汰：

- **volatile-random**：随机淘汰设置了过期时间的任意键值；
- **volatile-ttl**：优先淘汰更早过期的键值。
- **volatile-lru**（Redis3.0 之前，默认的内存淘汰策略）：淘汰所有设置了过期时间的键值中，最久未使用的键值；
- **volatile-lfu**（Redis 4.0 后新增的内存淘汰策略）：淘汰所有设置了过期时间的键值中，最少使用的键值；

在所有数据范围内进行淘汰：

- **allkeys-random**：随机淘汰任意键值;
- **allkeys-lru**：淘汰整个键值中最久未使用的键值；
- **allkeys-lfu**（Redis 4.0 后新增的内存淘汰策略）：淘汰整个键值中最少使用的键值























# 面试专栏



## Zset为什么用跳表而不用红黑树

简单翻译一下，主要是从内存占用、对范围查找的支持、实现难易程度这三方面总结的原因：

- 它们不是非常内存密集型的。基本上由你决定。改变关于节点具有给定级别数的概率的参数将使其比 btree 占用更少的内存。
- Zset 经常需要执行 ZRANGE 或 ZREVRANGE 的命令，即作为链表遍历跳表。通过此操作，跳表的缓存局部性至少与其他类型的平衡树一样好。
- 它们更易于实现、调试等。例如，由于跳表的简单性，我收到了一个补丁（已经在Redis master中），其中扩展了跳表，在 O(log(N) 中实现了 ZRANK。它只需要对代码进行少量修改。

我再详细补充点：

- **从内存占用上来比较，跳表比平衡树更灵活一些**。平衡树每个节点包含 2 个指针（分别指向左右子树），而跳表每个节点包含的指针数目平均为 1/(1-p)，具体取决于参数 p 的大小。如果像 Redis里的实现一样，取 p=1/4，那么平均每个节点包含 1.33 个指针，比平衡树更有优势。
- **在做范围查找的时候，跳表比平衡树操作要简单**。在平衡树上，我们找到指定范围的小值之后，还需要以中序遍历的顺序继续寻找其它不超过大值的节点。如果不对平衡树进行一定的改造，这里的中序遍历并不容易实现。而在跳表上进行范围查找就非常简单，只需要在找到小值之后，对第 1 层链表进行若干步的遍历就可以实现。
- **从算法实现难度上来比较，跳表比平衡树要简单得多**。平衡树的插入和删除操作可能引发子树的调整，逻辑复杂，而跳表的插入和删除只需要修改相邻节点的指针，操作简单又快速。



## Redis中LRU算法是怎样实现的





Redis 实现的是一种**近似 LRU 算法**，目的是为了更好的节约内存，它的**实现方式是在 Redis 的对象结构体中添加一个额外的字段，用于记录此数据的最后一次访问时间**。



当 Redis 进行内存淘汰时，会使用**随机采样的方式来淘汰数据**，它是随机取 5 个值（此值可配置），然后**淘汰最久没有使用的那个**。





Redis 实现的 LRU 算法的优点：

- 不用为所有的数据维护一个大链表，节省了空间占用；
- 不用在每次数据访问时都移动链表项，提升了缓存的性能；







但是 LRU 算法有一个问题，**无法解决缓存污染问题**，比如应用一次读取了大量的数据，而这些数据只会被读取这一次，那么这些数据会留存在 Redis 缓存中很长一段时间，造成缓存污染。

因此，在 Redis 4.0 之后引入了 LFU 算法来解决这个问题。







## Redis中LFU算法是怎样实现的

**什么是 LFU 算法？**



> LFU 全称是 Least Frequently Used 翻译为**最近最不常用的**，LFU 算法是根据数据访问次数来淘汰数据的，它的核心思想是“如果数据过去被访问多次，那么将来被访问的频率也更高”。
>
> 所以， LFU 算法会记录每个数据的访问次数。当一个数据被再次访问时，就会增加该数据的访问次数。这样就解决了偶尔被访问一次之后，数据留存在缓存中很长一段时间的问题，相比于 LRU 算法也更合理一些。





**Redis 是如何实现 LFU 算法的？**





LFU 算法相比于 LRU 算法的实现，多记录了「数据的访问频次」的信息。Redis 对象的结构如下：

```c
typedef struct redisObject {
    ...
      
    // 24 bits，用于记录对象的访问信息
    unsigned lru:24;  
    ...
} robj;
```



Redis 对象头中的 lru 字段，在 LRU 算法下和 LFU 算法下使用方式并不相同。

**在 LRU 算法中**，Redis 对象头的 24 bits 的 lru 字段是用来记录 key 的访问时间戳，因此在 LRU 模式下，Redis可以根据对象头中的 lru 字段记录的值，来比较最后一次 key 的访问时间长，从而淘汰最久未被使用的 key。

**在 LFU 算法中**，Redis对象头的 24 bits 的 lru 字段被分成两段来存储，高 16bit 存储 ldt(Last Decrement Time)，用来记录 key 的访问时间戳；低 8bit 存储 logc(Logistic Counter)，用来记录 key 的访问频次。



## LRU与LFU

LRU是最近最少使用页面置换算法(Least Recently Used),也就是首先淘汰最长时间未被使用的页面!

LFU是最近最不常用页面置换算法(Least Frequently Used),也就是淘汰一定时期内被访问次数最少的页!





## **Redis如何判断某个key应该在哪个实例？**



**Redis 集群的数据分片是redis进行分布式存储的一种，它引入了hash槽的概念，每个redis节点存储一定范围的hash槽 ，redis集群有16384个hash槽，每个key通过CRC16校验后对16384取模来决定存储在哪个槽哪个节点**



- 将16384个插槽分配到不同的实例
- 根据key的有效部分计算哈希值，对16384取余
- 余数作为插槽，寻找插槽所在实例即可

## Redis数据结构的使用场景



**持久化**



新数据结构

### 常用

**String**:SDS动态字符串



**Hash**：压缩列表或者哈希表（字典）



**list**：双向链表或者压缩列表



**set**：哈希表或者整数集合



**Zset**：哈希表和跳表





### 新数据结构



**BitMap**

是一串连续的二进制数组（0和1）



签到统计，356个位代表356天是否登录，登录置为1，没有还是0

判断用户登录







**HyperLogLog**

统计就是指统计一个集合中不重复的元素个数，占用内存很少

说 HyperLogLog **提供不精确的去重计数**。（HyperLogLog 是统计规则是基于概率完成的，不是非常准确，标准误算率是 0.81%。0

**每个 HyperLogLog 键只需要花费 12 KB 内存，就可以计算接近 `2^64` 个不同元素的基数**，和元素越多就越耗费内存的 Set 和 Hash 类型相比，HyperLogLog 就非常节省空间。



**GEO**

存储地理位置信息，并对存储的信息进行操作。



直接使用了 Sorted Set 集合类型（ZSET）





**Stream**



## Redis两种持久化比较







## Redis为什么设计为单线程



我们都知道单线程的程序是无法利用服务器的多核 CPU 的，那么早期 Redis 版本的主要工作（网络 I/O 和执行命令）为什么还要使用单线程呢？



**CPU并不是制约Redis性能表现的瓶颈所在**，更多情况下是受到**内存大小和网络I/O的限制**，所以单线程更加简单，也**避免了多线程的上下文切换和CPU消耗**，避免了增加复杂度、同时可能存在线程切换、甚至加锁和死锁造成的性能消耗。



所以 Redis 核心网络模型使用单线程并没有什么问题，如果你想要使用服务的多核CPU，可以在一台服务器上启动多个节点或者采用分片集群的方式。







### Redis6.0开辟了多线程

**在 Redis 6.0 版本之后，也采用了多个 I/O 线程来处理网络请求（写库与读库还是多线程）**，**这是因为随着网络硬件的性能提升，Redis 的性能瓶颈有时会出现在网络 I/O 的处理上**。







## Redis有几种集群模式

答：三种

- 主从模式
- 集群模式
- **cluster哨兵模式**







redis cluster在设计的时候，就考虑到了去中心化，去中间件，也就是说，集群中的每个节点都是平等的关系，都是对等的，每个节点都保存各自的数据和整个集群的状态。每个节点都和其他所有节点连接，而且这些连接保持活跃，这样就保证了我们只需要连接集群中的任意一个节点，就可以获取到其他节点的数据。
那么redis 是如何合理分配这些节点和数据的呢？
Redis 集群没有并使用传统的一致性哈希来分配数据，而是采用另外一种叫做哈希槽 (hash slot)的方式来分配的。redis cluster 默认分配了 16384 个slot，当我们set一个key 时，会用CRC16算法来取模得到所属的slot，然后将这个key 分到哈希槽区间的节点上，具体算法就是：CRC16(key) % 16384。



## Redis大key问题

查询和操作大key时，Redis影响性能，耗时



先找到大key -- bigkeys命令



使用redis-rdb-tools离线分析工具来扫描RDB持久化文件，虽然实时性略差，但是完全离线对性能无影响。



***使用 SCAN 命令查找大 key***



***使用 RdbTools 工具查找大 key***



### 删除大key



### ***1、分批次删除***

对于**删除大 Hash**，使用 `hscan` 命令，每次获取 100 个字段，再用 `hdel` 命令，每次删除 1 个字段。

Python代码：





### ***2、异步删除***



从 Redis 4.0 版本开始，可以采用**异步删除**法，**用 unlink 命令代替 del 来删除**。

这样 Redis 会将这个 key 放入到一个异步线程中进行删除，这样不会阻塞**主线程。**

除了主动调用 unlink 命令实现异步删除之外，我们还可以通过配置参数，达到某些条件的时候自动进行异步删除。











## Redis统计日活跃



### bitmap

用户登录时，使用setbit命令和用户id（假设id=123456）标记当日（2020-10-05）用户已经登录，具体命令如下：



（1）登录标记

用户登录时，使用setbit命令和用户id（假设id=123456）标记当日（2020-10-05）用户已经登录，具体命令如下：

```scss
# 时间复杂度O(1)
setbit login:20201005 123456 1
```



（2）每日用户登录数量统计

```scss
# 时间复杂度O(N)
bitcount login:20201005
```

（3）活跃用户（连续三日登录）统计

如果我们想要获取近三日活跃用户数量的话，可以使用bitop命令，
bitmap的bitop命令支持对bitmap进行`AND(与)`，`(OR)或`，`XOR(亦或)`，`NOT(非)`四种相关操作，我们对近三日的bitmap做`AND`操作即可，操作之后会形成一个新的bitmap，我们可以取名为`login:20201005:b3`

```scss
# 时间复杂度O(N)
bitop and login:20201005:b3 login:20201005 login:20201004 login:20201003
```

然后我们可以对`login:20201005:b3`使用bitcount或者getbit命令，用于统计活跃用户数量，或者查看某个用户是否为活跃用户



























**内存占用很小**















1、bitmap

优势是：非常均衡的特性，精准统计，可以得到每个统计对象的状态，秒出。
缺点是：当你的统计对象数量十分十分巨大时，可能会占用到一点存储空间，但也可在接受范围内。也可以通过分片，或者压缩的额外手段去解决。

2、HyperLogLog

优势是：可以统计夸张到无法想象的数量，并且占用小的夸张的内存。
缺点是：建立在牺牲准确率的基础上，而且无法得到每个统计对象的状态。





#### 统计用户的登录天数

**SETBIT  key1  0  1**
**SETBIT  key1  3  1**
**SETBIT  key1  8  1**
**SETBIT  key1  15  1**
**STRLEN key1**









可以将每一个key作为一个用户，然后每一个二进制位作为日期，用户登录时将对应日期的二进制位设置为1；如key1的二进制为：

10010000 10000001

则代表key1在第1、4、9、16天的时候有登录

BITCOUNT key1 0 0 结果为2则说明在1到8天的时候，key1的登录次数为2次；其他的以此类推。



（如果需要统计2到7天，可以先统计出1到8天的结果，然后再扣除第1天和第8天的结果；以及其他场景时，可根据具体的需求进行扣减或添加。）





















#### 统计活跃用户



用BITOP将day1、day2进行**或运算**，将结果赋值给day，查询day中1出现的次数



**BITOP  or day day1 day2** 
**STRLEN day  
BITCOUNT day 0 -1**

















### HyperLogLog

**必看**：https://cloud.tencent.com/developer/article/1975704



redis从2.8.9之后增加了HyperLogLog数据结构。这个数据结构，根据redis的官网介绍，这是一个概率数据结构，用来估算数据的基数。能通过牺牲准确率来减少内存空间的消耗。
HyperLogLog的方法





> 首先想到的方案是使用redis的set数据结构，因为它是一个无序集合，我们得到ip地址，然后存入set中即可实现统计与去重的效果，但是set有一个很大的问题是，每一条数据占用的空间会比较大，如果数据量很大的话可能会导致内存问题。
> 因此想到用一些比较节约空间的数据结构，想到了之前了解过的bitmap，空间占用比较低，不过bitmap比较适合预先知道用户数量的场景，我们知道总的用户数就知道到底需要定义一个多少容量的bitmap。而当前场景统计的ip总数是不确定的，因此bitmap不适用。
> 最终发现Redis的Hyperloglog数据结构非常的符合，Hyperloglog可以实现海量数据的统计与去重。与bitmap不同，它只是输入元素来计算基数，而不会存储元素本身。而这次需求不需要存储具体ip的值，只需要统计整体数量，并去重即可。













首先要说明，HyperLogLog实际上不会存储每个元素的值，它使用的是概率算法，通过存储元素的hash值的第一个1的位置，来计算元素数量。这样做存在误差，不适合绝对准确计数的场景。
redis中实现的HyperLogLog，只需要12K内存，在标准误差0.81%的前提下，能够统计2的64次方个数据。






- 统计一个 `APP` 的日活、月活数；
- 统计一个页面的每天被多少个不同账户访问量（Unique Visitor，UV））；
- 统计用户每天搜索不同词条的个数；
- 统计注册 IP 数。







HyperLogLog是一种算法，不会存储真实元数据，只会记录数量，很省空间。

有多个桶





## Zset实现直播间排行榜





https://blog.csdn.net/qq_34203492/article/details/98202848











































































