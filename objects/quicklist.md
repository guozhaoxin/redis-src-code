注：quicklist 部分根据 redis 源码中 3.2.0 进行解析，commit id 为 7ca8fbabe .



## 快表是什么

list 是 redis 最基础的数据类型之一；它的优点和缺点很明显：优点是它的双端操作很方便，但是每个节点都需要 prev 和 next 指针维持链表，而且节点内存是随机分布的，可能会导致明显的内存碎片，空间局部性也不太好。

在 redis 3.2 中引入了快表这种数据结构，它是 list 和 zip 两者结合的产物，它整体是一个 list，而每个节点则是一个 zip 节点。

![quicklist-struct](/Users/guozhaoxin/code/redis-src-code/objects/picture/quicklist-struct.jpg)

## 快表结构体

```c
//quicklist.h
typedef struct quicklistNode {
    struct quicklistNode *prev;
    struct quicklistNode *next;
    unsigned char *zl; //取决于节点位置，可能指向一个真正的 ziplist，也可能指向一个压缩后的 ziplist
    unsigned int sz;             /* ziplist size in bytes */ //zl 指向的 ziplist 大小，如果 ziplist 被压缩，也是压缩前的大小。
    unsigned int count : 16;     /* count of items in ziplist */
    unsigned int encoding : 2;   /* RAW==1 or LZF==2 */ // ziplist 是否被压缩以及压缩算法，固定为 1 没有压缩，2 使用 LZF 压缩。
    unsigned int container : 2;  /* NONE==1 or ZIPLIST==2 */ // 暂时没用，值固定为 2.
    unsigned int recompress : 1; /* was this node previous compressed? */ 
    unsigned int attempted_compress : 1; /* node can't compress; too small */
    unsigned int extra : 10; /* more bits to steal for future usage */
} quicklistNode;

typedef struct quicklistLZF {
    unsigned int sz; /* LZF size in bytes*/ // compressed 占用的总大小。
    char compressed[];
} quicklistLZF;

typedef struct quicklist {
    quicklistNode *head;
    quicklistNode *tail;
    unsigned long count;        /* total count of all entries in all ziplists */
    unsigned int len;           /* number of quicklistNodes */
    int fill : 16;              /* fill factor for individual nodes */ // ziplist 的大小，正负表示的意义不同。
    unsigned int compress : 16; /* depth of end nodes not to compress;0=off */ // 表示两端各自有几个节点不用压缩。
} quicklist;

typedef struct quicklistIter {
    const quicklist *quicklist;
    quicklistNode *current;
    unsigned char *zi; 
    long offset; /* offset in current ziplist */
    int direction;
} quicklistIter;

typedef struct quicklistEntry {
    const quicklist *quicklist;
    quicklistNode *node;
    unsigned char *zi;
    unsigned char *value;
    unsigned int sz;
    long long longval;
    int offset;
} quicklistEntry;
```

