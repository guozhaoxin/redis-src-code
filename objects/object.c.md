## object.c 源码

### 创建 robj 

```c
robj *createObject(int type, void *ptr) {
    robj *o = zmalloc(sizeof(*o));
    o->type = type;
    o->encoding = OBJ_ENCODING_RAW; // 为什么这里设置成raw string？
    o->ptr = ptr;
    o->refcount = 1; // 直接给计数 1

    /* Set the LRU to the current lruclock (minutes resolution). */
    o->lru = LRU_CLOCK();
    return o;
}
```

这个函数接收两个参数，第一个参数 type 表示对象的类型，即 robj 的 type 字段；第二个表示 robj 指向的底层实际数据，即 robj 的 ptr 字段。

函数本身比较小，**唯一的困惑在于为什么 encoding 会直接设置为 raw 类型**。



### 创建字符串对象

没什么可讲的，这里就是使用不同的函数生成字符串的几种不同对象。

```c
robj *createRawStringObject(const char *ptr, size_t len) {
    return createObject(OBJ_STRING,sdsnewlen(ptr,len));
}

/* Create a string object with encoding OBJ_ENCODING_EMBSTR, that is
 * an object where the sds string is actually an unmodifiable string
 * allocated in the same chunk as the object itself. */
robj *createEmbeddedStringObject(const char *ptr, size_t len) {
    robj *o = zmalloc(sizeof(robj)+sizeof(struct sdshdr8)+len+1); // 使用的是sds8类型，5类型太短了;正常的 sds8 长度是是可调的，这里因为实际表示 embstr，所以实际的数据部分是固定的。
    struct sdshdr8 *sh = (void*)(o+1); // 因为 sds 和 object 临接

    o->type = OBJ_STRING;
    o->encoding = OBJ_ENCODING_EMBSTR;
    o->ptr = sh+1;
    o->refcount = 1;
    o->lru = LRU_CLOCK();

    sh->len = len;
    sh->alloc = len;
    sh->flags = SDS_TYPE_8;
    if (ptr) {
        memcpy(sh->buf,ptr,len);
        sh->buf[len] = '\0';
    } else {
        memset(sh->buf,0,len+1);
    }
    return o;
}

/* Create a string object with EMBSTR encoding if it is smaller than
 * REIDS_ENCODING_EMBSTR_SIZE_LIMIT, otherwise the RAW encoding is
 * used.
 *
 * The current limit of 39 is chosen so that the biggest string object
 * we allocate as EMBSTR will still fit into the 64 byte arena of jemalloc. */
#define OBJ_ENCODING_EMBSTR_SIZE_LIMIT 44
robj *createStringObject(const char *ptr, size_t len) {
    if (len <= OBJ_ENCODING_EMBSTR_SIZE_LIMIT)
        return createEmbeddedStringObject(ptr,len);
    else
        return createRawStringObject(ptr,len);
}

robj *createStringObjectFromLongDouble(long double value, int humanfriendly) {
    char buf[256];
    int len;

    if (isinf(value)) {
        /* Libc in odd systems (Hi Solaris!) will format infinite in a
         * different way, so better to handle it in an explicit way. */
        if (value > 0) {
            memcpy(buf,"inf",3);
            len = 3;
        } else {
            memcpy(buf,"-inf",4);
            len = 4;
        }
    } else if (humanfriendly) {
        /* We use 17 digits precision since with 128 bit floats that precision
         * after rounding is able to represent most small decimal numbers in a
         * way that is "non surprising" for the user (that is, most small
         * decimal numbers will be represented in a way that when converted
         * back into a string are exactly the same as what the user typed.) */
        len = snprintf(buf,sizeof(buf),"%.17Lf", value);
        /* Now remove trailing zeroes after the '.' */
        if (strchr(buf,'.') != NULL) {
            char *p = buf+len-1;
            while(*p == '0') {
                p--;
                len--;
            }
            if (*p == '.') len--;
        }
    } else {
        len = snprintf(buf,sizeof(buf),"%.17Lg", value);
    }
    return createStringObject(buf,len);
}

```

### 创建 set 对象

```c
robj *createSetObject(void) {
    dict *d = dictCreate(&setDictType,NULL);
    robj *o = createObject(OBJ_SET,d);
    o->encoding = OBJ_ENCODING_HT;
    return o;
}

robj *createIntsetObject(void) {
    intset *is = intsetNew();
    robj *o = createObject(OBJ_SET,is);
    o->encoding = OBJ_ENCODING_INTSET;
    return o;
}
```

这里比较奇怪，set 本身包含整数集合和哈希两种实现，但它直接给哈希实现的函数取名为 createSetObject，而整数集合的则起名为 createIntsetObject。



### 创建哈希对象

```c
robj *createHashObject(void) {
    unsigned char *zl = ziplistNew();
    robj *o = createObject(OBJ_HASH, zl);
    o->encoding = OBJ_ENCODING_ZIPLIST;
    return o;
}
```

这个函数也是，用压缩列表创建哈希对象。但是这个 .c 文件中没有通过字典创建哈希对象的函数。



### 创建有序集合对象

```c
robj *createZsetObject(void) {
    zset *zs = zmalloc(sizeof(*zs));
    robj *o;

    zs->dict = dictCreate(&zsetDictType,NULL);
    zs->zsl = zslCreate();
    o = createObject(OBJ_ZSET,zs);
    o->encoding = OBJ_ENCODING_SKIPLIST;
    return o;
}

robj *createZsetZiplistObject(void) {
    unsigned char *zl = ziplistNew();
    robj *o = createObject(OBJ_ZSET,zl);
    o->encoding = OBJ_ENCODING_ZIPLIST;
    return o;
}
```

同样两个函数，第一个是跳表实现的有序集合；第二个是压缩表实现的有序集合。



### 创建列表

这个代码中似乎没有创建列表的代码？？？



### 释放 robj 指向的结构

redis 根据需要释放其指针指向的底层数据，要注意，这里各个函数只释放 ptr 指向的内存区域，而 robj 指针本身指向的区域不会管，可能是处于类型转换的考虑。

#### 释放字符串类型

```c
void freeStringObject(robj *o) {
    if (o->encoding == OBJ_ENCODING_RAW) {
        sdsfree(o->ptr);
    }
}
```

这里只有 raw 类型是需要释放的；int 类型本身就在 ptr 部分，没法释放；embstr 类型与 robj 结构体相连，且大小不变，释放没有意义。



#### 释放列表类型

```c
void freeListObject(robj *o) {
    switch (o->encoding) {
    case OBJ_ENCODING_QUICKLIST:
        quicklistRelease(o->ptr);
        break;
    default:
        serverPanic("Unknown list encoding type");
    }
}
```

看起来 3.2.5 中的列表只有快表一种实现方式。



#### 释放集合类型

```c
void freeSetObject(robj *o) {
    switch (o->encoding) {
    case OBJ_ENCODING_HT:
        dictRelease((dict*) o->ptr); // 显式转换
        break;
    case OBJ_ENCODING_INTSET:
        zfree(o->ptr);
        break;
    default:
        serverPanic("Unknown set encoding type");
    }
}
```

这里需要主要的是，两种不同类型的释放方式还是有区别的；

对于 dict 类型来说，需要释放字典结构体以及字典中的每一项，所以会有一个明确的类型强转；

对于整数集合来说，数据和头部分相连，直接用 free 释放掉了，不需要强转；



#### 释放有序集合

```c
void freeZsetObject(robj *o) {
    zset *zs;
    switch (o->encoding) {
    case OBJ_ENCODING_SKIPLIST:
        zs = o->ptr;
        dictRelease(zs->dict);
        zslFree(zs->zsl);
        zfree(zs);
        break;
    case OBJ_ENCODING_ZIPLIST:
        zfree(o->ptr);
        break;
    default:
        serverPanic("Unknown sorted set encoding");
    }
}
```



#### 释放哈希对象

```c
void freeHashObject(robj *o) {
    switch (o->encoding) {
    case OBJ_ENCODING_HT:
        dictRelease((dict*) o->ptr);
        break;
    case OBJ_ENCODING_ZIPLIST:
        zfree(o->ptr);
        break;
    default:
        serverPanic("Unknown hash encoding type");
        break;
    }
}
```

很类似于集合对象的释放。



### robj 引用计数

#### 增加计数

```c
void incrRefCount(robj *o) {
    o->refcount++;
}
void decrRefCountVoid(void *o) {
    decrRefCount(o); // 烦躁 什么时候需要明确声明指针类型？这里调用的 incrRefCount 接收的参数可是明确要求用 robj 的。
}
```



#### 减小计数

```c
void decrRefCount(robj *o) {
    if (o->refcount <= 0) serverPanic("decrRefCount against refcount <= 0");
    if (o->refcount == 1) {
        switch(o->type) {
        case OBJ_STRING: freeStringObject(o); break;
        case OBJ_LIST: freeListObject(o); break;
        case OBJ_SET: freeSetObject(o); break;
        case OBJ_ZSET: freeZsetObject(o); break;
        case OBJ_HASH: freeHashObject(o); break;
        default: serverPanic("Unknown object type"); break;
        }
        zfree(o);
    } else {
        o->refcount--;
    }
}
```

减小时，如果计数为 0，需要释放底层数据；根据不同的 robj 类型，选择不同的方法释放；但这里有个疑问，redis 开始时会直接创建短整数类型避免过多对象创建，那这些预制的对象是如何避免被错杀的，或者是这些对象天生自带额外的一个计数，所以永远不会出现计数为 0 的情况？



### 类型操作

#### 检查发送方类型是否正确

```c
int checkType(client *c, robj *o, int type) {
    if (o->type != type) {
        addReply(c,shared.wrongtypeerr);
        return 1;
    }
    return 0;
}
```

属实没想到，在这里会调用 client ，还是没整明白，为什么 objetc.c 会调用 server.c 的东西，难道 c 允许循环调用吗？



#### 返回具体类型

```c
char *strEncoding(int encoding) {
    switch(encoding) {
    case OBJ_ENCODING_RAW: return "raw";
    case OBJ_ENCODING_INT: return "int";
    case OBJ_ENCODING_HT: return "hashtable";
    case OBJ_ENCODING_QUICKLIST: return "quicklist";
    case OBJ_ENCODING_ZIPLIST: return "ziplist";
    case OBJ_ENCODING_INTSET: return "intset";
    case OBJ_ENCODING_SKIPLIST: return "skiplist";
    case OBJ_ENCODING_EMBSTR: return "embstr";
    default: return "unknown";
    }
}
```



#### 检查是字符串是否能换为 int

```c
int isObjectRepresentableAsLongLong(robj *o, long long *llval) {
    serverAssertWithInfo(NULL,o,o->type == OBJ_STRING);
    if (o->encoding == OBJ_ENCODING_INT) {
        if (llval) *llval = (long) o->ptr;
        return C_OK;
    } else {
        return string2ll(o->ptr,sdslen(o->ptr),llval) ? C_OK : C_ERR;
    }
}
```

检查是否能把 robj 表示的数据转换为一个 long long。



#### 优化字符串存储

```c
robj *tryObjectEncoding(robj *o) {
    long value;
    sds s = o->ptr;
    size_t len;

    /* Make sure this is a string object, the only type we encode
     * in this function. Other types use encoded memory efficient
     * representations but are handled by the commands implementing
     * the type. */
    serverAssertWithInfo(NULL,o,o->type == OBJ_STRING);

    /* We try some specialized encoding only for objects that are
     * RAW or EMBSTR encoded, in other words objects that are still
     * in represented by an actually array of chars. */
    if (!sdsEncodedObject(o)) return o;

    /* It's not safe to encode shared objects: shared objects can be shared
     * everywhere in the "object space" of Redis and may end in places where
     * they are not handled. We handle them only as values in the keyspace. */
     if (o->refcount > 1) return o;

    /* Check if we can represent this string as a long integer.
     * Note that we are sure that a string larger than 20 chars is not
     * representable as a 32 nor 64 bit integer. */
    len = sdslen(s);
    if (len <= 20 && string2l(s,len,&value)) {
        /* This object is encodable as a long. Try to use a shared object.
         * Note that we avoid using shared integers when maxmemory is used
         * because every object needs to have a private LRU field for the LRU
         * algorithm to work well. */
        if ((server.maxmemory == 0 ||
             (server.maxmemory_policy != MAXMEMORY_VOLATILE_LRU &&
              server.maxmemory_policy != MAXMEMORY_ALLKEYS_LRU)) &&
            value >= 0 &&
            value < OBJ_SHARED_INTEGERS)
        {
            decrRefCount(o); // gzx? 当允许共享对象时，可以这样使用，但是这里先减少再增加会不会导致对象被错误释放？还是共享数组永远不可能为1？
            incrRefCount(shared.integers[value]);
            return shared.integers[value];
        } else {
            if (o->encoding == OBJ_ENCODING_RAW) sdsfree(o->ptr);
            o->encoding = OBJ_ENCODING_INT;
            o->ptr = (void*) value;
            return o;
        }
    }

    /* If the string is small and is still RAW encoded,
     * try the EMBSTR encoding which is more efficient.
     * In this representation the object and the SDS string are allocated
     * in the same chunk of memory to save space and cache misses. */
    if (len <= OBJ_ENCODING_EMBSTR_SIZE_LIMIT) {
        robj *emb;

        if (o->encoding == OBJ_ENCODING_EMBSTR) return o;
        emb = createEmbeddedStringObject(s,sdslen(s));
        decrRefCount(o);
        return emb;
    }

    /* We can't encode the object...
     *
     * Do the last try, and at least optimize the SDS string inside
     * the string object to require little space, in case there
     * is more than 10% of free space at the end of the SDS string.
     *
     * We do that only for relatively large strings as this branch
     * is only entered if the length of the string is greater than
     * OBJ_ENCODING_EMBSTR_SIZE_LIMIT. */
    if (o->encoding == OBJ_ENCODING_RAW &&
        sdsavail(s) > len/10)
    {
        o->ptr = sdsRemoveFreeSpace(o->ptr);
    }

    /* Return the original object. */
    return o;
}
```

这个函数主要用来优化字符串对象，比如将字符串尝试转化为 int 或者将 raw 转为 embstr，或者压缩 embstr 的空间；



### 对象时间操作

#### 返回对象空转时间

```c
unsigned long long estimateObjectIdleTime(robj *o) {
    unsigned long long lruclock = LRU_CLOCK();
    if (lruclock >= o->lru) {
        return (lruclock - o->lru) * LRU_CLOCK_RESOLUTION;
    } else {
        return (lruclock + (LRU_CLOCK_MAX - o->lru)) * // gzx? 什么情况下才会出现大于？？？？
                    LRU_CLOCK_RESOLUTION;
    }
}
```



#### 不改变对象时间戳查询对象

```c
robj *objectCommandLookup(client *c, robj *key) {
    dictEntry *de;

    if ((de = dictFind(c->db->dict,key->ptr)) == NULL) return NULL;
    return (robj*) dictGetVal(de);
}
```

这个函数主要是当执行 objects 命令时，可以不改变对象的时间戳而又能得到结果。
