# Info

版本信息：Redis 6.0



# 对象数据结构

```c
typedef struct redisObject {
    // 对象类型
    unsigned type: 4;
  	// 对象编码
    unsigned encoding: 4;
    // 正常占32位。LRU_BITS指定这个字段占24位
    // 记录redis对象的访问时间(lru)或者访问频率(lfu)，取决于redis的淘汰策略
    unsigned lru: LRU_BITS;
    // 引用计数，表示当前对象被多少个地方引用
    int refcount;
    // redis->type的实际实现对象
    void *ptr;
} robj;
```

redis的数据结构设计非常省内存，并且让结构体按8字节对齐，避免CPU访问内存时多次跨cache line。



## 对象类型

定义对象的类型

```c
#define OBJ_STRING 0    /* 字符串对象 */
#define OBJ_LIST 1      /* 列表对象 */
#define OBJ_SET 2       /* 集合对象 */
#define OBJ_ZSET 3      /* 有序集合对象 */
#define OBJ_HASH 4      /* 哈希对象 */
#define OBJ_MODULE 5    /* 模块对象 */
#define OBJ_STREAM 6    /* 流对象 */
```



## 对象编码

对象实际的编码类型

```c
#define OBJ_ENCODING_RAW 0     /* 原始类型 */
#define OBJ_ENCODING_INT 1     /* 整数 */
#define OBJ_ENCODING_HT 2      /* 哈希表 */
#define OBJ_ENCODING_ZIPMAP 3  /* 压缩map */
#define OBJ_ENCODING_LINKEDLIST 4 /* 此版本不用，老版本list的编码 */
#define OBJ_ENCODING_ZIPLIST 5 /* 压缩列表 */
#define OBJ_ENCODING_INTSET 6  /* 整数集合 */
#define OBJ_ENCODING_SKIPLIST 7  /* 跳表 */
#define OBJ_ENCODING_EMBSTR 8  /* 内嵌结构，string对象会和object存在同一块连续地址 */
#define OBJ_ENCODING_QUICKLIST 9 /* 压缩列表的链表 */
#define OBJ_ENCODING_STREAM 10 /* listpacks的平衡树 */
```



## lru

这个字段是给内存淘汰策略用的，当内存不足时，需要把一部分对象从内存中淘汰出去（或者不淘汰，取决于配置的淘汰策略）。

记录的是这个对象最近的访问时间 或者 这个对象的访问频率（保存什么值取决于淘汰策略）



## refcount

引用计数。redis的内存是自己管理，需要统计这个对象是否有被引用，方便回收内存。

相互引用问题，redis对象没有相互引用的情况，无需考虑这个问题



## ptr

ptr是一个指针，指向这个对象实际的数据结构



# string对象

string对象有两种编码类型：`EmbeddedString` 和 `RawString`

- EmbeddedString

  EmbeddedString是针对string小对象的优化，使得redisObject整个对象的字节数小于或等于64字节，小对象只需要申请一次内存，并且整个对象能够整个放入到cache line中，提高性能

- RawString

  与EmbeddedString不同，RawString的redisObject对象和sds对象的内存申请是分开的



## 数据结构

不管是`EmbeddedString`还是`RawString`，string对象用的数据结构都是一样的，区别只在于分配的内存地址是不是跟redisObject连续的

```c
struct __attribute__ ((__packed__)) sdshdr8 {
    // 字符串已使用的长度
    uint8_t len;
    // 字符串分配的空间
    uint8_t alloc;
    // 只使用到低3位
    unsigned char flags;
    // 实际保存字符串的内存
    char buf[];
};
```

字符串定义了多个sds（sdshdr8，sdshdr16，sdshdr32，sdshdr64）类型，主要针对不同的字符串大小，来针对`len`和`alloc`字段，使用不同的字节大小的int类型。

len：表示分配的buf已使用的内存空间

alloc：表示分配的buf申请的内存空间

flags：表示使用的是哪种sds类型：sdshdr8，sdshdr16，sdshdr32，sdshdr64

buf：指向实际保存字符串的内存空间



