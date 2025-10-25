# Info

version：redis 6.0



# inset

intset保存是一批有序的整数集数组。根据保存数值的大小，使用不同的编码来保存这一批数据

有序集合的内存分配也是刚好够用，不会多分配。

编码一旦升级，就不会降回去了

# 数据结构

```c
typedef struct intset {
    // 编码
    uint32_t encoding;
    // 数组长度
    uint32_t length;
    // 保存的内容
    int8_t contents[];
} intset;
```



- encoding：定义该集合的编码。有三种类型值

  ```c
  #define INTSET_ENC_INT16 (sizeof(int16_t))		// 2字节
  #define INTSET_ENC_INT32 (sizeof(int32_t))		// 4字节
  #define INTSET_ENC_INT64 (sizeof(int64_t))		// 8字节
  ```

- length：定义该数组的元素个数

- contents：保存着元素的内容，一个元素所占用的字节数取决于encoding



# 添加元素

```c
intset *intsetAdd(intset *is, int64_t value, uint8_t *success) {
    // 获取当前值的编码类型
    uint8_t valenc = _intsetValueEncoding(value);
    uint32_t pos;
    // 默认是插入操作
    if (success) *success = 1;

    // 如果新的值编码所占用的空间大小 大于 旧值所占用的空间大小
    if (valenc > intrev32ifbe(is->encoding)) {
        /* This always succeeds, so we don't need to curry *success. */
        // 原集合中所有数组都要升级
        return intsetUpgradeAndAdd(is,value);
    } else {
        // 在集合中搜索这个元素值，并把位置读取出来
        if (intsetSearch(is,value,&pos)) {
            // 更新操作，success置为0
            if (success) *success = 0;
            return is;
        }

        is = intsetResize(is,intrev32ifbe(is->length)+1);
        // 元素往后挪，给插入的元素让位置
        if (pos < intrev32ifbe(is->length)) intsetMoveTail(is,pos,pos+1);
    }

    // 设置插入元素
    _intsetSet(is,pos,value);
    // 更新长度
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
    return is;
}
```

intset是一个严格升序的集合。元素的添加分成两种情况：

- 编码类型发生升级。比如int16变成int32，这个时候所有元素所占用的空间都变大了，整个数组要重新调整位置
- 编码类型保持不变。更新或者新增值





- 编码类型发生升级

  当编码类型发生升级时，因为intset是升序，这个新加进来的值要么是最大值，要么是最小值（并且不会跟原有对象有重复），所以可以直接根据value值的正负选择插入到第一位或者最后一位

  ```c
  static intset *intsetUpgradeAndAdd(intset *is, int64_t value) {
      // 当前编码值
      uint8_t curenc = intrev32ifbe(is->encoding);
      // 新的编码值
      uint8_t newenc = _intsetValueEncoding(value);
      // 当前的数组长度
      int length = intrev32ifbe(is->length);
      // 当前值是正数还是负数。正数的话新值插入到尾部，负数的话新值插入到头部
      int prepend = value < 0 ? 1 : 0;
  
      is->encoding = intrev32ifbe(newenc);
      // 调整intset集合的大小
      is = intsetResize(is,intrev32ifbe(is->length)+1);
  
      // 从后往前更新，不会覆盖到原值
      // 1. 根据旧编码当前到当前length位置的值
      // 2. 将旧值的编码大小改变，然后重新设置到intset
      // 3. value是负数往下标后一位挪，value是正数保持原下标不动
      while(length--)
          _intsetSet(is,length+prepend,_intsetGetEncoded(is,length,curenc));
  
      // value是负数，设置到头部
      if (prepend)
          _intsetSet(is,0,value);
      else
          _intsetSet(is,intrev32ifbe(is->length),value);      // value是负数，设置到尾部
      // 更新数组的长度
      is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
      return is;
  }
  ```

- 编码类型不变

  编码类型不变，有可能在原集合中有相同的值，这个时候就啥也不做。否则，找到插入位置，将插入位置的元素往后移，放入到插入位置

  用二分查找搜索插入位置

  ```c
  intset *intsetAdd(intset *is, int64_t value, uint8_t *success) {
      // 获取当前值的编码类型
      uint8_t valenc = _intsetValueEncoding(value);
      uint32_t pos;
      // 默认是插入操作
      if (success) *success = 1;
  
      // 如果新的值编码所占用的空间大小 大于 旧值所占用的空间大小
      if (valenc > intrev32ifbe(is->encoding)) {
          ……
      } else {		// 编码类型不变的情况
          // 在集合中搜索这个元素值，并把位置读取出来
          if (intsetSearch(is,value,&pos)) {
              // 更新操作，success置为0
              if (success) *success = 0;
              return is;
          }
  
          is = intsetResize(is,intrev32ifbe(is->length)+1);
          // 元素往后挪，给插入的元素让位置
          if (pos < intrev32ifbe(is->length)) intsetMoveTail(is,pos,pos+1);
      }
  
      // 设置插入元素
      _intsetSet(is,pos,value);
      // 更新长度
      is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
      return is;
  }
  ```



# 删除元素

```c
intset *intsetRemove(intset *is, int64_t value, int *success) {
    // 获取当前value的编码
    uint8_t valenc = _intsetValueEncoding(value);
    uint32_t pos;
    if (success) *success = 0;

    // 小优化：编码值 大于当前集合的编码值的话，肯定不在集合中，也就不用搜素了
    // 二分查找元素，然后将后面的元素往前挪，直接覆盖掉
    if (valenc <= intrev32ifbe(is->encoding) && intsetSearch(is,value,&pos)) {
        uint32_t len = intrev32ifbe(is->length);

        /* We know we can delete */
        if (success) *success = 1;

        /* Overwrite value with tail and update length */
        if (pos < (len-1)) intsetMoveTail(is,pos+1,pos);
        // 调整集合大小
        is = intsetResize(is,len-1);
        is->length = intrev32ifbe(len-1);
    }
    return is;
}
```

