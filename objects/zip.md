## 压缩列表

压缩列表虽然名为列表，但它与我们正常使用的列表有很大区别；我们一般使用的列表，每个元素在列表中占用的字节数是固定的，因此查找和更改指定索引处的元素会很快；但是处理元素大小可变的情况，如果使用列表，往往不得不按照最极端的情况设置单一元素的大小，这时就非常容易出现各种碎片的情况；redis 中的压缩列表是一种很紧凑的数据结构，它每个列表元素占用的大小是不固定的，单一元素大小会进行存储，但这就使得按索引获取元素以及插入、删除等操作变得更复杂。

### 结构体定义

#### 列表整体结构体定义

下面是压缩列表的整体结构。

![redis-ziplist-structure](./picture/redis-ziplist-structure.png)

- zlbytes ，32 位无符号整数，表示当前 ziplist 占用的总字节数；

- zltail，32 位无符号整数，ziplist 最后一个 entry 的头相对于 ziplist 最开始的偏移量，通过它，不需要完全遍历 ziplist 就可以直接跳转到最后一项；

- zllen，16 位 无符号整数，ziplist 的 entry 数量；当 zllen 比 2**16-2 大时，zllen数值不准确，需要完全遍历 entry 列表来获取 entry 的总数目；

- zlend，单字节的特殊值，等于255，标识着 ziplist 结束；

- prev_entry_len, 压缩表中，每一项都会存储前一项 entry 的长度，这部分可能占 1 字节，可能占 5 字节；当前面一项 entry 长小于 254 时，占用 1 字节；否则占用 5 字节，其中第 1 个字节固定为 254，实际长度用后面 4 字节表示；之所以是 254 而不是 255，因为 255 在跳表中表示 zlend；通过这个属性值，再加上 zltail,  压缩表中可以很方便的实现逆序遍历；

- encoding,  可能占用 1、2、5 字节：

  | 编码                                                | 占用字节数 | 含义                                                         |
  | --------------------------------------------------- | ---------- | ------------------------------------------------------------ |
  | 00bbbbbb                                            | 1 字节     | value 表示长度不超过 63 字节的字节数组                       |
  | 01bbbbbb xxxxxxxx                                   | 2 字节     | value 表示长度不超过 16383 字节的字节数组                    |
  | 10\_\_\_\_\_\__ aaaaaaaa bbbbbbbb cccccccc dddddddd | 5 字节     | value 表示长度不超过 2<sup>32</sup> - 1 字节的字节数组       |
  | 11000000                                            | 1 字节     | value 表示 int16_t 整数                                      |
  | 11010000                                            | 1 字节     | value 表示 int32_t 整数                                      |
  | 11100000                                            | 1 字节     | value 表示 int64_t 整数                                      |
  | 11110000                                            | 1 字节     | value 表示 24 位有符号整数                                   |
  | 11111110                                            | 1 字节     | value 表示 8 位有符号整数                                    |
  | 1111xxxx                                            | 1 字节     | 此时没有 content，值存储在 xxxx 部分；而且只存储 0001-1101, 表示 0-12. |

- value，这一项 entry 实际的内容；实际内容由 encoding 进行解释；prev_entry_len、encoding、value 共同组成一项 entry；

#### 列表元素

```c
typedef struct zlentry {
    unsigned int prevrawlensize, prevrawlen;
    unsigned int lensize, len;
    unsigned int headersize;
    unsigned char encoding;
    unsigned char *p;
} zlentry;
```

上面是单一元素的定义，但要注意，这个结构体并不与上面 entry 各字段对应，而是为了方便对压缩表进行操作，各字段含义如下：

- prevrawlensize, 是指 prev_entry_len 占用字节的大小，它的值只有 1 和 5；；
- prevrawlen,  前一节点实际占用的字节数量； 
- lensize, 编码本项 entry 长度所需的字节大小，它的值有 1 2 和 5 三种情况;
- len, 当前项 value 占用字节数;
- headersize, 当前节点大小，prevrawlensize+lensize，它主要是对压缩表每一项进行操作时，方便计算；
- encoding, 节点编码方式；
- p, 指向节点的指针，即对应 entry 的 prev_entry_len 开始的地方；



### 压缩表操作

#### 索引元素

由于压缩表是中元素紧凑、每个元素长度不定，导致按照索引获取元素时，就需要全结构遍历。压缩表中索引元素时，支持负索引，此时是按照从压缩表尾往头数的，源码如下：

```c
unsigned char *ziplistIndex(unsigned char *zl, int index) {
    unsigned char *p;
    unsigned int prevlensize, prevlen = 0;
    if (index < 0) {
        index = (-index)-1;
        p = ZIPLIST_ENTRY_TAIL(zl);
        if (p[0] != ZIP_END) {
            ZIP_DECODE_PREVLEN(p, prevlensize, prevlen);
            while (prevlen > 0 && index--) {
                p -= prevlen;
                ZIP_DECODE_PREVLEN(p, prevlensize, prevlen); // 宏定义会解析当前 entry 的字段，求出前一项的字节数。
            }
        }
    } else {
        p = ZIPLIST_ENTRY_HEAD(zl); // 获取到头 entry 的位置
        while (p[0] != ZIP_END && index--) {
            p += zipRawEntryLength(p);  // 函数返回当前 entry 占用的字节总数，包括 prev、encoding、content 三部分各自长度
        }
    }
    return (p[0] == ZIP_END || index > 0) ? NULL : p;
}
```



#### 获取元素数量

压缩表中的 zllength 字段，是用来记录 entry 数量的，但是它只占用 2 字节，所以如果 entry 数量超过一定值，就需要完整遍历压缩表才能知道了：

```c
unsigned int ziplistLen(unsigned char *zl) {
    unsigned int len = 0;
    if (intrev16ifbe(ZIPLIST_LENGTH(zl)) < UINT16_MAX) { // zllength 没溢出，可以直接读取并返回；
        len = intrev16ifbe(ZIPLIST_LENGTH(zl));
    } else {
        unsigned char *p = zl+ZIPLIST_HEADER_SIZE; // 开始遍历压缩表
        while (*p != ZIP_END) {
            p += zipRawEntryLength(p);
            len++;
        }

        /* Re-store length if small enough */
        if (len < UINT16_MAX) ZIPLIST_LENGTH(zl) = intrev16ifbe(len); // 如果遍历结果显示数量不大，需要更新回去。
    }
    return len;
}
```



#### 插入元素

```c
// ziplist.c
static unsigned char *__ziplistInsert(unsigned char *zl, unsigned char *p, unsigned char *s, unsigned int slen) {
    size_t curlen = intrev32ifbe(ZIPLIST_BYTES(zl)), reqlen;
    unsigned int prevlensize, prevlen = 0;
    size_t offset;
    int nextdiff = 0;
    unsigned char encoding = 0;
    long long value = 123456789; /* initialized to avoid warning. Using a value
                                    that is easy to see if for some reason
                                    we use it uninitialized. */
    zlentry tail;
	// 第一步，计算出插入位置前面一项的长度
    /* Find out prevlen for the entry that is inserted. */
    if (p[0] != ZIP_END) {
        ZIP_DECODE_PREVLEN(p, prevlensize, prevlen); // 求出 p 指向的 entry 的前一项的长度等信息
    } else { // p 指向压缩表的 最后一个字节
        unsigned char *ptail = ZIPLIST_ENTRY_TAIL(zl);
        if (ptail[0] != ZIP_END) { // 说明此时压缩表是非空的，至少有一个 entry
            prevlen = zipRawEntryLength(ptail);
        }
    }
	
    // 设置 encoding 类型
    /* See if the entry can be encoded */
    if (zipTryEncoding(s,slen,&value,&encoding)) {
        /* 'encoding' is set to the appropriate integer encoding */
        reqlen = zipIntSize(encoding);
    } else {
        /* 'encoding' is untouched, however zipEncodeLength will use the
         * string length to figure out how to encode it. */
        reqlen = slen;
    }
    
    // 计算新的 entry 的总字节数
    /* We need space for both the length of the previous entry and
     * the length of the payload. */
    reqlen += zipPrevEncodeLength(NULL,prevlen); // 表示前一项长度需要的字节数
    reqlen += zipEncodeLength(NULL,encoding,slen); // 表示当前项 encoding 等需要占用的字节数

    /* When the insert position is not equal to the tail, we need to
     * make sure that the next entry can hold this entry's length in
     * its prevlen field. */
    nextdiff = (p[0] != ZIP_END) ? zipPrevLenByteDiff(p,reqlen) : 0;

    /* Store offset because a realloc may change the address of zl. */
    offset = p-zl;
    zl = ziplistResize(zl,curlen+reqlen+nextdiff);
    p = zl+offset;

    /* Apply memory move when necessary and update tail offset. */
    if (p[0] != ZIP_END) {
        /* Subtract one because of the ZIP_END bytes */
        memmove(p+reqlen,p-nextdiff,curlen-offset-1+nextdiff);

        /* Encode this entry's raw length in the next entry. */
        zipPrevEncodeLength(p+reqlen,reqlen);

        /* Update offset for tail */
        ZIPLIST_TAIL_OFFSET(zl) =
            intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+reqlen);

        /* When the tail contains more than one entry, we need to take
         * "nextdiff" in account as well. Otherwise, a change in the
         * size of prevlen doesn't have an effect on the *tail* offset. */
        tail = zipEntry(p+reqlen);
        if (p[reqlen+tail.headersize+tail.len] != ZIP_END) {
            ZIPLIST_TAIL_OFFSET(zl) =
                intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+nextdiff);
        }
    } else {
        /* This element will be the new tail. */
        ZIPLIST_TAIL_OFFSET(zl) = intrev32ifbe(p-zl);
    }

    /* When nextdiff != 0, the raw length of the next entry has changed, so
     * we need to cascade the update throughout the ziplist */
    if (nextdiff != 0) {
        offset = p-zl;
        zl = __ziplistCascadeUpdate(zl,p+reqlen);
        p = zl+offset;
    }

    /* Write the entry */
    p += zipPrevEncodeLength(p,prevlen);
    p += zipEncodeLength(p,encoding,slen);
    if (ZIP_IS_STR(encoding)) {
        memcpy(p,s,slen);
    } else {
        zipSaveInteger(p,value,encoding);
    }
    ZIPLIST_INCR_LENGTH(zl,1);
    return zl;
}
```

压缩表添加元素时，支持表头、表尾插入，这个函数中的 p 参数可以是表头，也可以是表尾，上层调用函数指定；s 为要插入的元素，而 slen 表示 s 的长度。

此外需要注意的是，s 作为要插入的内容，我们并不知道它表示的到底是一个字节数组，还是一个整数值，函数内部会调用 zipTryEncoding 尝试将其转为整数，如果可以转换成功，就认为 s 是个整数，否则认为是个字节数组；

整个函数的逻辑如下：![ziplist-insert](./picture/ziplist-insert.png)

这个函数需要注意的地方是，因为插入新的数据，可能导致调整新项后面的 entry 的 prevlensize 变化；代码通过一个 nextdiff 变量来记录是否需要调整。



#### 删除元素

```c
// ziplist.c
static unsigned char *__ziplistDelete(unsigned char *zl, unsigned char *p, unsigned int num) {
    unsigned int i, totlen, deleted = 0;
    size_t offset;
    int nextdiff = 0;
    zlentry first, tail;

    first = zipEntry(p);
    for (i = 0; p[0] != ZIP_END && i < num; i++) {
        p += zipRawEntryLength(p);
        deleted++;
    }

    totlen = p-first.p;
    if (totlen > 0) {
        if (p[0] != ZIP_END) {
            /* Storing `prevrawlen` in this entry may increase or decrease the
             * number of bytes required compare to the current `prevrawlen`.
             * There always is room to store this, because it was previously
             * stored by an entry that is now being deleted. */
            nextdiff = zipPrevLenByteDiff(p,first.prevrawlen);
            p -= nextdiff; // 假设需要扩容，则返回 4，p 要左移，占用部分前一项的空间；假设需要缩容，则返回 -4，p 要右移，释放部分自己的空间；
            zipPrevEncodeLength(p,first.prevrawlen);

            /* Update offset for tail */
            ZIPLIST_TAIL_OFFSET(zl) =
                intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))-totlen); // 为什么只减去 totlen？nextdiff 不处理吗？

            /* When the tail contains more than one entry, we need to take
             * "nextdiff" in account as well. Otherwise, a change in the
             * size of prevlen doesn't have an effect on the *tail* offset. */
            tail = zipEntry(p);
            if (p[tail.headersize+tail.len] != ZIP_END) {
                ZIPLIST_TAIL_OFFSET(zl) =
                   intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+nextdiff); // 扩容时，需要增加字节；缩容时，需要减小字节；
            }

            /* Move tail to the front of the ziplist */
            memmove(first.p,p,
                intrev32ifbe(ZIPLIST_BYTES(zl))-(p-zl)-1);
        } else {
            /* The entire tail was deleted. No need to move memory. */
            ZIPLIST_TAIL_OFFSET(zl) =
                intrev32ifbe((first.p-zl)-first.prevrawlen); // 新的 tail 节点是 first 前面的节点。
        }

        /* Resize and update length */
        offset = first.p-zl; // 因为下面会重新分配内存，导致 first.p 指向的内存不可用，所以必须先计算出偏移量并保存，方便后续操作。
        zl = ziplistResize(zl, intrev32ifbe(ZIPLIST_BYTES(zl))-totlen+nextdiff);
        ZIPLIST_INCR_LENGTH(zl,-deleted);
        p = zl+offset;

        /* When nextdiff != 0, the raw length of the next entry has changed, so
         * we need to cascade the update throughout the ziplist */
        if (nextdiff != 0)
            zl = __ziplistCascadeUpdate(zl,p);
    }
    return zl;
}
```

这个函数指定要删除的元素的起始位置，以及连续删除的数量。

整个函数的逻辑与插入比较像，同样需要考虑是否要连续调整多个节点的 prev。

有一点要注意的是，被删除的最后一个节点之后的那个节点(上面代码中的 p)，它现在的 prev 表示被删除的第一个节点前面的那一项，即上面代码中 first 的前一项；代码中通过调整 p 的左移或者右移，来调整此时 p 的prev，最后把 p 开始到 ZLEND 的整个空间都左移到 p 起始的位置即可。关键是 p 的左右移动会不会发生溢出？只要分三种情况考虑即可，假设 first 的前面一项是 x，然后被删除的节点只有 first 一项：

##### case 1

假设 x 比较短，其大小只要 1 个字节表示，则 first 的 prev 只要 1 个字节；如果 first 本身也比较短，其长度用 1 个字节表示即可，则 p 的 prev 只要 1 个字节；当 first 被删除后，nextdiff 为 0， p 不需要移动；

#####  case 2

基本同上，但 first 比较长，假设其长度大小 1 字节表示不了，此时 p 的 prev 必然用 5 字节表示；删除 first 后，p 的前一项变成了 x，prev 只要 1 字节表示，p 会向右移动 4 个字节，因为原来 p 的 prev 字段占用 5 字节，不会溢出；

##### case 3

假设 x 比较长，大小需要用 5 字节表示，则对 first 来说，它至少占用 5 + 1 + 1 = 7 个字节，即这段被删除的空间至少有 7 字节长，而 p 如果要左移，最多也就 4 个字节，也不会发生溢出；
