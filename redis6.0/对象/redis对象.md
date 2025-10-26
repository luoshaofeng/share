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





# 字符串对象

字符串对象的编码有三种：`OBJ_ENCODING_INT` ， `OBJ_ENCODING_RAW` 和 `OBJ_ENCODING_EMBSTR`



看`set`命令对`val`的编码

- OBJ_ENCODING_INT

  当字符串是一个整数，并且长度小于20时。ptr指向的是一个个long long类型的整数

否则，就按照`OBJ_ENCODING_RAW`和`OBJ_ENCODING_EMBSTR`的编码规则

```c
void setCommand(client *c) {
    int j;
    robj *expire = NULL;
    int unit = UNIT_SECONDS;
    int flags = OBJ_SET_NO_FLAGS;

    ……

    // 编码value的值，有三种编码：OBJ_ENCODING_RAW、OBJ_ENCODING_INT 和 OBJ_ENCODING_EMBSTR
    c->argv[2] = tryObjectEncoding(c->argv[2]);
  
    ……
}

// 编码字符串类型
robj *tryObjectEncoding(robj *o) {
    long value;
    sds s = o->ptr;
    size_t len;

    ……
      
    len = sdslen(s);
    // 长度小于20 && 可以转成long类型
    if (len <= 20 && string2l(s,len,&value)) {
        // 数值的长度 在[0,10000]之间，复用这个对象（也是OBJ_ENCODING_INT对象），直接返回
        if ((server.maxmemory == 0 ||
            !(server.maxmemory_policy & MAXMEMORY_FLAG_NO_SHARED_INTEGERS)) &&
            value >= 0 &&
            value < OBJ_SHARED_INTEGERS)
        {
            decrRefCount(o);        // 原对象的引用-1，方便内存释放
            incrRefCount(shared.integers[value]);       // 共享对象的引用+1
            return shared.integers[value];      // 返回共享对象
        } else {
            // 不能复用共享对象，将编码修改为 OBJ_ENCODING_INT
            if (o->encoding == OBJ_ENCODING_RAW) {
                // 释放掉原有的对象内存
                sdsfree(o->ptr);
                o->encoding = OBJ_ENCODING_INT;
                o->ptr = (void*) value;
                return o;
            } else if (o->encoding == OBJ_ENCODING_EMBSTR) {        // 如果是OBJ_ENCODING_EMBSTR这种编码
                decrRefCount(o);
                // 根据long long对象创建string对象
                return createStringObjectFromLongLongForValue(value);
            }
        }
    }

    // 字符串不能 转成 数字类型，判断字符串的长度大小
    if (len <= OBJ_ENCODING_EMBSTR_SIZE_LIMIT) {
        robj *emb;
        // 如果原来就是OBJ_ENCODING_EMBSTR编码，直接返回
        if (o->encoding == OBJ_ENCODING_EMBSTR) return o;
        // 创建OBJ_ENCODING_EMBSTR编码对象
        emb = createEmbeddedStringObject(s,sdslen(s));
        decrRefCount(o);
        return emb;
    }
  
  	……
    
    return o;
}
```





# 列表对象

列表对象的编码只有一种：`OBJ_ENCODING_QUICKLIST`



查看`lpush`命令

```c
void pushGenericCommand(client *c, int where) {
    int j, pushed = 0;

    ……

    robj *lobj = lookupKeyWrite(c->db,c->argv[1]);

    if (lobj && lobj->type != OBJ_LIST) {
        addReply(c,shared.wrongtypeerr);
        return;
    }

  	// LPUSH key element [element ...]
    for (j = 2; j < c->argc; j++) {
        if (!lobj) {
            lobj = createQuicklistObject();
            quicklistSetOptions(lobj->ptr, server.list_max_ziplist_size,
                                server.list_compress_depth);
            dbAdd(c->db,c->argv[1],lobj);
        }
      	// 保存元素
        listTypePush(lobj,c->argv[j],where);
        pushed++;
    }
    
  	……
}


void listTypePush(robj *subject, robj *value, int where) {
    // 编码必须是 OBJ_ENCODING_QUICKLIST
    if (subject->encoding == OBJ_ENCODING_QUICKLIST) {
        ……
    } else {
        serverPanic("Unknown list encoding");
    }
}
```







# 集合对象

集合对象的编码有两种：`OBJ_ENCODING_INTSET` 和 `OBJ_ENCODING_HT`

如果加入的元素都是整数，那么使用`OBJ_ENCODING_INTSET`。否则，使用`OBJ_ENCODING_HT`



查看`sadd`命令

添加元素时，如果全部是整数时，编码使用`OBJ_ENCODING_INTSET`；否则，使用`OBJ_ENCODING_HT`。**移除元素时，编码不会发生转换**

- 创建set对象

  ```c
  robj *setTypeCreate(sds value) {
    	// 创建intset对象
      if (isSdsRepresentableAsLongLong(value,NULL) == C_OK)
          return createIntsetObject();
    	// 创建dictht对象
      return createSetObject();
  }
  ```

- 添加set对象

  ```c
  void saddCommand(client *c) {
      robj *set;
      int j, added = 0;
  
      set = lookupKeyWrite(c->db,c->argv[1]);
      if (set == NULL) {
        	// 创建set对象
          set = setTypeCreate(c->argv[2]->ptr);
          dbAdd(c->db,c->argv[1],set);
      } else {
        	// 错误返回
          if (set->type != OBJ_SET) {
              addReply(c,shared.wrongtypeerr);
              return;
          }
      }
  
      // SADD key member [member ...]
      for (j = 2; j < c->argc; j++) {
          if (setTypeAdd(set,c->argv[j]->ptr)) added++;
      }
      ……
  }
  
  
  int setTypeAdd(robj *subject, sds value) {
      long long llval;
      if (subject->encoding == OBJ_ENCODING_HT) {
          dict *ht = subject->ptr;
          dictEntry *de = dictAddRaw(ht,value,NULL);
          ……
      } else if (subject->encoding == OBJ_ENCODING_INTSET) {
          // 当添加的元素不是整数时，转为OBJ_ENCODING_HT对象
          if (isSdsRepresentableAsLongLong(value,&llval) == C_OK) {
              uint8_t success = 0;
              subject->ptr = intsetAdd(subject->ptr,llval,&success);
              ……
          } else {
              // 设置成OBJ_ENCODING_HT编码
              setTypeConvert(subject,OBJ_ENCODING_HT);
  
              serverAssert(dictAdd(subject->ptr,sdsdup(value),NULL) == DICT_OK);
              return 1;
          }
      } else {
          serverPanic("Unknown set encoding");
      }
      return 0;
  }
  ```

  



# 有序集合对象

有序集合的编码对象有两种：`OBJ_ENCODING_ZIPLIST`和`OBJ_ENCODING_SKIPLIST`

默认使用`OBJ_ENCODING_ZIPLIST`，满足一定条件会转换成`OBJ_ENCODING_SKIPLIST`



在有序集合中，`OBJ_ENCODING_SKIPLIST`编码对应的是zset结构，是用dict对象和zskiplist对象组合使用的

```c
typedef struct zset {
    // 字典对象
    dict *dict;
    // 跳表
    zskiplist *zsl;
} zset;
```

当用ziplist保存时，`ele`在前，`score`在后





查看`zadd`命令

- 当保存值的长度超过了`zset_max_ziplist_value`或者`zset_max_ziplist_entries`的值为0，使用`zset`对象。否则，使用`ziplist`对象
- `zset_max_ziplist_value`：`zset`数据结构中使用`ziplist`结构保存时，`value`的最大长度
- `zset_max_ziplist_entries`：zset数据结构中使用`ziplist`结构保存时，`entry`的最大个数

```c
void zaddGenericCommand(client *c, int flags) {
    ……
   	
    zobj = lookupKeyWrite(c->db, key);
    if (zobj == NULL) {
        if (xx) goto reply_to_client;
        if (server.zset_max_ziplist_entries == 0 ||
            server.zset_max_ziplist_value < sdslen(c->argv[scoreidx + 1]->ptr)) {
          	// 创建zset对象
            zobj = createZsetObject();
        } else {
          	// 创建zset ziplist对象
            zobj = createZsetZiplistObject();
        }
        dbAdd(c->db, key, zobj);
    } else {
        if (zobj->type != OBJ_ZSET) {
            ……
        }
    }

    for (j = 0; j < elements; j++) {
        double newscore;
        score = scores[j];
        int retflags = flags;

        ele = c->argv[scoreidx + 1 + j * 2]->ptr;
      	// 添加元素
        int retval = zsetAdd(zobj, score, ele, &retflags, &newscore);
        ……
    }

		……
}
```

- 创建zset对象

  - 创建两个对象：`dict`和`zskiplist`

  ```c
  robj *createZsetObject(void) {
      zset *zs = zmalloc(sizeof(*zs));
      robj *o;
  
      // 创建字典对象
      zs->dict = dictCreate(&zsetDictType,NULL);
      // 创建跳表对象
      zs->zsl = zslCreate();
      o = createObject(OBJ_ZSET,zs);
      o->encoding = OBJ_ENCODING_SKIPLIST;
      return o;
  }
  ```

- 创建zset ziplist对象

  - 创建ziplist对象

  ```c
  robj *createZsetZiplistObject(void) {
      unsigned char *zl = ziplistNew();
      robj *o = createObject(OBJ_ZSET,zl);
      o->encoding = OBJ_ENCODING_ZIPLIST;
      return o;
  }
  ```

- 添加元素

  如果当前编码对象是`OBJ_ENCODING_ZIPLIST`，当ziplist保存的entry对(key,val)超过了`zset_max_ziplist_entries` 或者 元素的长度超过了`zset_max_ziplist_value` 或者 整个`ziplist`所占用的字节数过大，那么就会将`ziplist`转为`skiplist`

  ```c
  int zsetAdd(robj *zobj, double score, sds ele, int *flags, double *newscore) {
      ……
  
      if (zobj->encoding == OBJ_ENCODING_ZIPLIST) {
          unsigned char *eptr;
          // 找到元素
          if ((eptr = zzlFind(zobj->ptr, ele, &curscore)) != NULL) {
              ……
          } else if (!xx) {       // 不是存在时写入，新增元素
              // 编码转换，当超过所限制的entry对数，或者value超过最大长度，或者整个ziplist entry数过大了（不安全）
              if (zzlLength(zobj->ptr) + 1 > server.zset_max_ziplist_entries ||
                  sdslen(ele) > server.zset_max_ziplist_value ||
                  !ziplistSafeToAdd(zobj->ptr, sdslen(ele))) {
                	// 转换成 OBJ_ENCODING_SKIPLIST
                  zsetConvert(zobj,OBJ_ENCODING_SKIPLIST);
              } else {
                  ……
              }
          } else {
              ……
          }
      }
  
      if (zobj->encoding == OBJ_ENCODING_SKIPLIST) {
          ……
      } else {
          serverPanic("Unknown sorted set encoding");
      }
    
      return 0; /* Never reached. */
  }



# 哈希对象

哈希对象的编码有两种：`OBJ_ENCODING_ZIPLIST`和`OBJ_ENCODING_HT`

默认使用`OBJ_ENCODING_ZIPLIST`，满足一定条件转成`OBJ_ENCODING_HT`

使用`OBJ_ENCODING_ZIPLIST`对象保存时，`field`在前，`value`在后



查看`hset`命令

```c
void hsetCommand(client *c) {
    int i, created = 0;
    robj *o;

    if ((c->argc % 2) == 1) {
        addReplyErrorFormat(c,"wrong number of arguments for '%s' command",c->cmd->name);
        return;
    }

  	// 查找或者创建对象
    if ((o = hashTypeLookupWriteOrCreate(c,c->argv[1])) == NULL) return;
  	// hash类型尝试转换
    hashTypeTryConversion(o,c->argv,2,c->argc-1);

    ……
}
```



- 创建对象

  创建hash对象时使用的是`OBJ_ENCODING_ZIPLIST`编码

  ```c
  robj *hashTypeLookupWriteOrCreate(client *c, robj *key) {
      robj *o = lookupKeyWrite(c->db,key);
      if (o == NULL) {
        	// 创建hash对象
          o = createHashObject();
          dbAdd(c->db,key,o);
      } else {
          if (o->type != OBJ_HASH) {
              addReply(c,shared.wrongtypeerr);
              return NULL;
          }
      }
      return o;
  }
  
  robj *createHashObject(void) {
      unsigned char *zl = ziplistNew();
      robj *o = createObject(OBJ_HASH, zl);
    	// 使用OBJ_ENCODING_ZIPLIST编码
      o->encoding = OBJ_ENCODING_ZIPLIST;
      return o;
  }

- 编码对象转换

  编码对象可能从`OBJ_ENCODING_ZIPLIST`转成`OBJ_ENCODING_HT`，需要满足其中一个条件：

  - 保存的键 或者 值的字符串长度超过`hash_max_ziplist_value`
  - 整个`ziplist`保存的键值长度超过了`ziplist`的最大字节限制

  ```c
  void hashTypeTryConversion(robj *o, robj **argv, int start, int end) {
      int i;
      size_t sum = 0;
  
      if (o->encoding != OBJ_ENCODING_ZIPLIST) return;
  
      for (i = start; i <= end; i++) {
          // 不是字符串编码则跳过
          if (!sdsEncodedObject(argv[i]))
              continue;
          // 计算字符串的长度
          size_t len = sdslen(argv[i]->ptr);
          // 长度大于 限制的最大长度
          if (len > server.hash_max_ziplist_value) {
              // 转成OBJ_ENCODING_HT对象
              hashTypeConvert(o, OBJ_ENCODING_HT);
              return;
          }
          sum += len;
      }
  
      // 如果整个ziplist的字节数过大，也转成 OBJ_ENCODING_HT
      if (!ziplistSafeToAdd(o->ptr, sum))
          hashTypeConvert(o, OBJ_ENCODING_HT);
  }
  ```

  



