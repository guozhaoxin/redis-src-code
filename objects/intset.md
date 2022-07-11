## 整数集合

整数集合是 redis 的底层数据结构之一，主要用于集合，如果集合中的所有元素都是整数时，redis 有可能使用整数集合作为底层结构。



### 结构体定义

```c
typedef struct intset {
    uint32_t encoding;
    uint32_t length;
    int8_t contents[];
} intset;
```

![intset-struct](./picture/intset-struct.png)

各字段含义如下：

- encoding, 表示集合中每个元素的类型；整数集合中每个元素占用字节数相同，但不同时刻该大小可能不同，该字段指明当前每个元素的大小，包括16 位、32  位、64 位，均为有符号类型；
- length, 表示集合中元素的数量；
- contents, 这是一个字节数组，虽然声明为 int8, 但其真正的意义需要根据 encoding 来解释；

需要注意以下几点：

- 整数集合中的元素是有序的，这样方便使用二分法进行查找；

  ![asc-intset](./picture/asc-intset.png)

- 如果新加入集合的元素使用的字节数比集合当前单元素占用的大，则集合会发生升级，每个元素占用的字节数均达到新元素的字节数量；

- 集合不会发生降级，即使所有元素都不需要这么多空间；

![intset-upgrade-downgrade](./picture/intset-upgrade-downgrade.png)



### 整数集合函数

#### _intsetGetEncoded

这个函数如下：

```c
static int64_t _intsetGetEncoded(intset *is, int pos, uint8_t enc) {
    int64_t v64;
    int32_t v32;
    int16_t v16;

    if (enc == INTSET_ENC_INT64) {
        memcpy(&v64,((int64_t*)is->contents)+pos,sizeof(v64)); 
        memrev64ifbe(&v64);// 处理大小端问题
        return v64;
    } else if (enc == INTSET_ENC_INT32) {
        memcpy(&v32,((int32_t*)is->contents)+pos,sizeof(v32));
        memrev32ifbe(&v32);
        return v32;
    } else {
        memcpy(&v16,((int16_t*)is->contents)+pos,sizeof(v16));
        memrev16ifbe(&v16);
        return v16;
    }
}
```

该函数获取集合中指定索引的数值，而且获取后还会处理大小端问题，整数集合中貌似元素的存储都是按照小端模式存储的；

**问题 如果是考虑数据的可移植性，为什么 sds 中看不到对 alloc 和 len 的大小端处理，而在 intset 中，除了 contents 部分外，连 encoding 和 length 都需要考虑大小端？？？？？**



#### intsetNew

```c
intset *intsetNew(void) {
    intset *is = zmalloc(sizeof(intset));
    is->encoding = intrev32ifbe(INTSET_ENC_INT16);// 为什么要处理大小端问题
    is->length = 0;
    return is;
}
```

生成一个新的集合对象，可以看到，初始时每个元素都是 2 字节。



#### intsetSearch

```c
static uint8_t intsetSearch(intset *is, int64_t value, uint32_t *pos) 
```

这个函数用来查找目标数据是否在集合中出现，二分查找的的使用例子。



#### intsetUpgradeAndAdd

```c
static intset *intsetUpgradeAndAdd(intset *is, int64_t value) {
    uint8_t curenc = intrev32ifbe(is->encoding);
    uint8_t newenc = _intsetValueEncoding(value);
    int length = intrev32ifbe(is->length);
    int prepend = value < 0 ? 1 : 0;

    /* First set new encoding and resize */
    is->encoding = intrev32ifbe(newenc);
    is = intsetResize(is,intrev32ifbe(is->length)+1);

    /* Upgrade back-to-front so we don't overwrite values.
     * Note that the "prepend" variable is used to make sure we have an empty
     * space at either the beginning or the end of the intset. */
    while(length--)
        _intsetSet(is,length+prepend,_intsetGetEncoded(is,length,curenc));

    /* Set the value at the beginning or the end. */
    if (prepend)
        _intsetSet(is,0,value);
    else
        _intsetSet(is,intrev32ifbe(is->length),value);
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
    return is;
}
```

集合大小升级并添加新元素。

这里比较有意思的地方在于，升级后，需要移动数据；而导致升级的数据一定是在新集合的头部或者尾部，因此这里使用从后往前迁移数据的方式，解决数据被覆盖的问题。



#### intsetAdd

```c
intset *intsetAdd(intset *is, int64_t value, uint8_t *success) 
```

向集合中添加数值的函数，



#### intsetRemove

```c
intset *intsetRemove(intset *is, int64_t value, int *success) 
```

从集合中删除一个数值，成功失败通过 success 指针指明。