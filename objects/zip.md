## 压缩列表

压缩列表虽然名为列表，但它与我们正常使用的列表有很大区别；我们一般使用的列表，每个元素在列表中占用的字节数是固定的，因此查找和更改指定索引处的元素时会很快；如果处理元素大小可变的情况，如果使用列表的话，往往不得不按照最极端的情况设置单一元素的大小，这时就非常容易出现各种碎片的情况；redis 中的压缩列表是一种很紧凑的数据结构，它每个列表元素占用的大小是不固定的，单一元素大小会进行存储，但这就使得按索引获取元素以及插入、删除等操作变得更复杂。

### 结构体定义

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

上面是单一元素的定义，各字段如下：

- prevrawlensize, 是指prevrawlen的大小，有1字节和5字节两种；（？？？）
- prevrawlen, 前一节点长度； 
- lensize, 编码len所需的字节大小;
- len, len为当前节点长度;
- headersize, 当前节点大小；
- encoding, 节点编码方式；
- p, 指向节点的指针（为什么还有指针呢？）；



#### 列表整体结构体定义

![redis-ziplist-structure](./picture/redis-ziplist-structure.png)

- zlbytes是一个无符号整数，表示当前ziplist占用的总字节数；
- zltail是ziplist最后一个entry的指针相对于ziplist最开始的偏移量，通过它，不需要完全遍历ziplist就可以对最后的entry进行操作；
- zllen是ziplist的entry数量。当zllen比2**16-2大时，需要完全遍历entry列表来获取entry的总数目；
- zlend是一个单字节的特殊值，等于255，标识着ziplist的内存结束点；