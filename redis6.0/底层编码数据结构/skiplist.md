# skiplist

# 数据结构

```c
typedef struct zskiplist {
    // 首节点 和 尾节点
    struct zskiplistNode *header, *tail;
    // 元素个数
    unsigned long length;
    // 跳表最大层数
    int level;
} zskiplist;

```

跳表的最外层结构，记录整个跳表的信息。

- `header`，`tail`：头节点和尾节点
- `length`：跳表中的元素个数
- `level`：整个跳表中的最高层级，最高层数为32。





**跳表节点**

```c
typedef struct zskiplistNode {
    // 元素值
    sds ele;
    // 得分
    double score;
    // 前一个节点
    struct zskiplistNode *backward;

    // 当前节点的多个层级
    struct zskiplistLevel {
        // 当前层级的下一个节点
        struct zskiplistNode *forward;
        // 跨度，两个节点之间的距离（相邻的两个节点跨度是1）
        unsigned long span;
    } level[];
} zskiplistNode;
```

- `ele`：sds(string)类型的一个元素值
- `score`：得分，用于排序，决定节点在哪个位置。当`score`相等，根据`ele`排序
- `backward`：指向前一个节点
- `zskiplistLevel`：指当前节点的层级，用于元素的随机跨越
  - `forward`：指向下一个在该层级的元素
  - `span`：两个节点之间的距离（相邻的两个节点跨度是1）



# 对象创建

- 空跳表层级是1
- 跳表有一个哨兵节点，层级是最高层级32级。不保存任何元素信息

```c
zskiplist *zslCreate(void) {
    int j;
    zskiplist *zsl;

    zsl = zmalloc(sizeof(*zsl));
    zsl->level = 1;
    zsl->length = 0;
    // 创建32层级节点
    // 这是一个哨兵节点
    zsl->header = zslCreateNode(ZSKIPLIST_MAXLEVEL, 0,NULL);
    // 每层级节点初始化
    for (j = 0; j < ZSKIPLIST_MAXLEVEL; j++) {
        zsl->header->level[j].forward = NULL;
        zsl->header->level[j].span = 0;
    }
    zsl->header->backward = NULL;
    zsl->tail = NULL;
    return zsl;
}
```





# 添加元素

当插入一个新节点时，会在初始化节点时取一个随机数，作为该节点的层级。当该节点的层级比整个跳表的层级要高时，更新跳表的层级。

- `update[i]`：待插入节点x，x插入到`update[i]`节点到`update[i].forward`节点之间

- `rank[i]`： `header`节点到`update[i]`节点之间的跨度，节点之间不一定是连续的，所以rank[i] <= rank[0]的

- `rank[0]`：最后一层的节点跨度。

- `rank[0] - rank[i]`：`update[i]`到`update[0]`节点的跨度

- `x节点`：`update[0]`的下一个节点，与`update[0]`之间的跨度为1

  原先，`update[i]`与`update[i].forward`的节点距离为`update[i]->level[i].span`，中间插入了`节点x`，那么`update[i]`与`x节点`的距离为`rank[0] - rank[i] + 1`，`x节点`与`update[i].forward节点`的跨度为`update[i]->level[i].span - (rank[0] - rank[i])`



简单来说，新增一个节点，这个节点的层数是随机的。然后从最高层更新，维护每一层之间的节点关系与跨度。

```c
zskiplistNode *zslInsert(zskiplist *zsl, double score, sds ele) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    unsigned int rank[ZSKIPLIST_MAXLEVEL];
    int i, level;

    serverAssert(!isnan(score));
    x = zsl->header;
    // 从哨兵节点的最高层开始查
    for (i = zsl->level - 1; i >= 0; i--) {
      	// 在该层级中，header节点到update[i]节点的跨度
        rank[i] = i == (zsl->level - 1) ? 0 : rank[i + 1];
        // 找到待插入节点的位置，插入到update[i]的后面
        // 得分相同的话，会根据元素去比较
        while (x->level[i].forward &&
               (x->level[i].forward->score < score ||
                (x->level[i].forward->score == score &&
                 sdscmp(x->level[i].forward->ele, ele) < 0))) {
            // 起点 到 当前节点 的跨度
            rank[i] += x->level[i].span;
            x = x->level[i].forward;
        }
        update[i] = x;
    }

    // 随机取一个层级
    level = zslRandomLevel();
    // 层级 大于 当前的最大层级
    if (level > zsl->level) {
      	// 高出当前跳表的层级，只能插入到头节点后面
        for (i = zsl->level; i < level; i++) {
          	// 头节点 到 头节点的距离是0
            rank[i] = 0;
          	// 指向哨兵节点，也就是头节点。新创建的节点在头节点后面
            update[i] = zsl->header;
            // 更新哨兵节点的跨度
            update[i]->level[i].span = zsl->length;
        }
        // 更新当前的最高层级
        zsl->level = level;
    }
    // 创建节点
    x = zslCreateNode(level, score, ele);

    for (i = 0; i < level; i++) {
        // 将当前x节点插入到update[i]节点和update[i].forward节点的中间
        x->level[i].forward = update[i]->level[i].forward;
        update[i]->level[i].forward = x;

        // 更新x节点和update[i]节点的跨度值
        x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);
        update[i]->level[i].span = (rank[0] - rank[i]) + 1;
    }

    // 高层级的跨度加1，新增了一个节点
  	// header节点的高层级指向NULL的话，保存的跨度值就是zsl.length
    for (i = level; i < zsl->level; i++) {
        update[i]->level[i].span++;
    }

    x->backward = (update[0] == zsl->header) ? NULL : update[0];
    // 更新关联前一个节点，相邻的两个节点。最低层级是1，forward肯定时相邻的下一个节点
    if (x->level[0].forward)
        x->level[0].forward->backward = x;
    else
        zsl->tail = x;
    // 节点数加1
    zsl->length++;
    return x;
}
```





# 删除元素

跳表的删除需要score和ele一起参与

- 找到每一层 待删除节点的上一个节点，将这个节点保存到`update[i]`
- 拿到待删除节点（如果存在的话）
- 删除这个节点，并且维护好每一层的`span`和`forward`关系

```c
int zslDelete(zskiplist *zsl, double score, sds ele, zskiplistNode **node) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    int i;

    x = zsl->header;
    // 找到每个层级的待删除节点的上一个层级节点
    for (i = zsl->level - 1; i >= 0; i--) {
        // 找到待删除节点的上一个层级节点
        while (x->level[i].forward &&
               (x->level[i].forward->score < score ||
                (x->level[i].forward->score == score &&
                 sdscmp(x->level[i].forward->ele, ele) < 0))) {
            x = x->level[i].forward;
        }
        // 待删除节点的上一个层级节点保存到update[i]
        update[i] = x;
    }
    // 在第一层，如果这个节点存在的话，那么就是这个节点
    x = x->level[0].forward;
    // 确实是这个节点
    if (x && score == x->score && sdscmp(x->ele, ele) == 0) {
        // 删掉这个节点
        zslDeleteNode(zsl, x, update);
        // 要不要把x节点返回赋值给node，node为NULL则不需要，释放x节点
        if (!node)
            zslFreeNode(x);
        else
            *node = x;
        return 1;
    }
    return 0;
}
```



# 查找元素

## 根据跨度查找（下标）

- 从当前跳表的最高层级开始找
- 如果当前遍历到的节点的总跨度（header到当前节点的跨度） + 与下一个节点的跨度 大于 要查找的跨度。那么就降低层级，继续比较。否则，就跳到下一个节点继续比较。直接刚好查到总跨度 与 查找的跨度相等。返回该节点

```c
zskiplistNode *zslGetElementByRank(zskiplist *zsl, unsigned long rank) {
    zskiplistNode *x;
    unsigned long traversed = 0;
    int i;

    x = zsl->header;
    // 从最高层级往下遍历
    for (i = zsl->level - 1; i >= 0; i--) {
        // 找到跨度刚好为rank的节点
        while (x->level[i].forward && (traversed + x->level[i].span) <= rank) {
            traversed += x->level[i].span;
            x = x->level[i].forward;
        }
        // 跨度刚好，直接返回
        if (traversed == rank) {
            return x;
        }
    }
    return NULL;
}
```



## 根据得分查找（范围查找）

- 根据跳表的首尾元素确认查找的元素在不在范围内（按照得分排序）
- 从最高层查找，找到范围内的第一个元素（最小值）
- 判断第一个元素是否超过了最大值

```c
zskiplistNode *zslFirstInRange(zskiplist *zsl, zrangespec *range) {
    zskiplistNode *x;
    int i;

    // 判断跳表是否有数据在区间内
    if (!zslIsInRange(zsl, range)) return NULL;

    x = zsl->header;
    for (i = zsl->level - 1; i >= 0; i--) {
        // 跳表查找最小值
        while (x->level[i].forward &&
               !zslValueGteMin(x->level[i].forward->score, range)) //大于等于区间最小值
            x = x->level[i].forward; // 小于最小值则更新x
    }

    // 拿到第一个在区间内的节点（大于最小值的第一个节点）
    x = x->level[0].forward;
    serverAssert(x != NULL);

    // 判断这个节点是否小于最大值
    if (!zslValueLteMax(x->score, range)) return NULL;
    return x;
}
```

