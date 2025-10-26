# Info

version：redis 6.0



# dictht

# 数据结构

```c
typedef struct dict {
    // 字典上的操作函数
    dictType *type;
    // 私有数据
    void *privdata;
    // 哈希表
    dictht ht[2];
    // -1表示未迁移
    long rehashidx;
    // safe模式下 表示当前是否正在迭代
    unsigned long iterators;
} dict;
```

字典上的主要字段有三个：`ht[2]`，`rehashidx`和`iterators`

- `rehashidx`：表示当前哈希表是否正在迁移，当值为-1时，表示未迁移。当`rehashidx`大于等于0时，表示当前哈希表的迁移进度
- `iterators`：哈希表的迭代（遍历）分成安全模式和非安全模式。当安全迭代时，iterators的值会被置为1，之后此哈希表就会被禁止rehash
- `ht[2]`：表示两个哈希表。正常情况下只会用到`ht[0]`，当哈希表扩容时，会把`ht[0]`的数据逐渐的迁移到`ht[1]`，`rehashidx`就表示`ht[0]`的迁移进度。当完全迁移完之后，`ht[1]`就会覆盖掉`ht[0]`，`rehashidx`被赋值为-1，回到正常情况



# 哈希表

对于哈希表来说，如果hash表正在重哈希，那么添加，查找，删除操作都会迁移桶数据

```c
typedef struct dictht {
    // 哈希元素数组
    dictEntry **table;
    // 桶数量
    unsigned long size;
    // 掩码
    unsigned long sizemask;
    // 表中的元素个数
    unsigned long used;
} dictht;

typedef struct dictEntry {
    void *key;

    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;

    // 链表，产生哈希冲突时用链表法
    struct dictEntry *next;
} dictEntry;
```

哈希表采用链表法来解决哈希冲突。

数据结构如上：

- `dictEntry`：哈希表上的一个元素
- `size`：哈希表的桶数量
- `sizemask`：值为size-1，掩码。用来辅助定位到某个key应该放到哪个桶中
- `used`：哈希表中已保存的元素个数



# 添加元素

哈希表添加元素时，如果元素已存在，那么则会返回错误。一般先查找，然后再根据查找的结果选择是更新还是新增。

- 字典上的`rehashidx`不等于-1，那么则会迁移这个桶的数据到新哈希表上
- 如果找到这个key对应的entry，那么`dictAddRaw`返回NULL，`dictAdd`就返回错误
- 如果正在rehash的话，往新表上添加元素

```c
int dictAdd(dict *d, void *key, void *val) {
    dictEntry *entry = dictAddRaw(d, key,NULL);

    if (!entry) return DICT_ERR;
    // 将值设置到entry中
    dictSetVal(d, entry, val);
    return DICT_OK;
}

dictEntry *dictAddRaw(dict *d, void *key, dictEntry **existing) {
    long index;
    dictEntry *entry;
    dictht *ht;

    // 是否在重哈希，在重哈希的话，迁移桶的位置
    if (dictIsRehashing(d)) _dictRehashStep(d);

    // 找到这个key对应的entry，只需要更新就行
    if ((index = _dictKeyIndex(d, key, dictHashKey(d, key), existing)) == -1)
        return NULL;

    // 如果是正在扩容，那么往新的哈希表上添加数据
    ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0];
    // 申请entry所需的空间
    entry = zmalloc(sizeof(*entry));
    // 使用头插入法添加entry
    entry->next = ht->table[index];
    ht->table[index] = entry;
    // 使用的元素数量+1
    ht->used++;

    // 将key设置到entry中
    dictSetKey(d, entry, key);
    return entry;
}
```



# 查找元素

- 如果当前哈希表正在rehash，迁移桶
- 现在`ht[0]`查找元素，如果找不到并且正在rehash的话，去`ht[1]`找

```c
dictEntry *dictFind(dict *d, const void *key) {
    dictEntry *he;
    uint64_t h, idx, table;

    // 字典是空的，直接返回nil
    if (dictSize(d) == 0) return NULL;
    // 迁移数据
    if (dictIsRehashing(d)) _dictRehashStep(d);
    //计算key的哈希值
    h = dictHashKey(d, key);
    for (table = 0; table <= 1; table++) {
        // 找出key对应的桶位置
        idx = h & d->ht[table].sizemask;
        // 遍历桶上的元素值
        he = d->ht[table].table[idx];
        while (he) {
            // 找到key对应的entry了，直接返回
            if (key == he->key || dictCompareKeys(d, key, he->key))
                return he;
            he = he->next;
        }
        // 没有重哈希的话，直接返回
        if (!dictIsRehashing(d)) return NULL;
    }
    return NULL;
}
```



# 删除元素

如果当前哈希表在rehash，那么迁移桶中的元素

定位到哈希表的位置，然后从链表中删除

```c
int dictDelete(dict *ht, const void *key) {
    return dictGenericDelete(ht, key, 0) ? DICT_OK : DICT_ERR;
}

static dictEntry *dictGenericDelete(dict *d, const void *key, int nofree) {
    uint64_t h, idx;
    dictEntry *he, *prevHe;
    int table;

    // 字典没值，则直接返回
    if (d->ht[0].used == 0 && d->ht[1].used == 0) return NULL;

    // 判断是否在重hash
    if (dictIsRehashing(d)) _dictRehashStep(d);
    // 拿到hash值
    h = dictHashKey(d, key);

    for (table = 0; table <= 1; table++) {
        idx = h & d->ht[table].sizemask;
        // 定位到key所在的hash位置
        he = d->ht[table].table[idx];
        prevHe = NULL;
        while (he) {
            //比较key是否相等
            if (key == he->key || dictCompareKeys(d, key, he->key)) {
                /* Unlink the element from the list */
                if (prevHe)
                    prevHe->next = he->next;
                else
                    d->ht[table].table[idx] = he->next;
                //延迟删还是直接删
                if (!nofree) {
                    dictFreeKey(d, he);
                    dictFreeVal(d, he);
                    zfree(he);
                }
                // 元素减1
                d->ht[table].used--;
                // 返回被删除的元素
                return he;
            }
            // 遍历链表
            prevHe = he;
            he = he->next;
        }
        // 当前没有重哈希，遍历第一个就可以了
        if (!dictIsRehashing(d)) break;
    }
    return NULL; /* not found */
}
```

