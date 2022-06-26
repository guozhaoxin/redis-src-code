## 列表定义

```c
// adlist.h
typedef struct listNode {
    struct listNode *prev;
    struct listNode *next;
    void *value;
} listNode;

typedef struct listIter {
    listNode *next;
    int direction;
} listIter;

typedef struct list {
    listNode *head;
    listNode *tail;
    void *(*dup)(void *ptr);
    void (*free)(void *ptr);
    int (*match)(void *ptr, void *key);
    unsigned long len;
} list;
```

链表中主要有三个结构体：

- listNode 定义了列表中每个节点的信息，可以看到这是个双向链表节点；
- list 定义了列表结构体，负责具体的对外工作，当然这个结构体也有一些辅助性的函数指针作为属性；
- listIter, 对链表进行遍历时候使用；



## 列表函数

### listCreate

#### 函数原型

```c
// adlist.c
list *listCreate(void)
```

这个函数主要负责生成一个新的 list 结构体，并初始化其头尾指针、数量，将几个函数指针均置为空。



### listRelease

#### 函数原型

```c
// adlist.c
void listRelease(list *list)
{
    unsigned long len;
    listNode *current, *next;

    current = list->head;
    len = list->len;
    while(len--) {
        next = current->next;
        if (list->free) list->free(current->value);
        zfree(current);
        current = next;
    }
    zfree(list);
}
```

这个函数用来释放列表中的每个节点，并在最后释放整个列表结构体。这里调用了结构体的 free 指针。



### listAddNodeHead

#### 函数原型

```c
// adlist.c
list *listAddNodeHead(list *list, void *value)
```

这个函数向列表头部添加节点，不过需要注意的是，虽然在操作过程中，list 本身不会更改内存区域，但函数还是返回了这个指针而不是 void。



### listAddNodeTail

#### 函数原型

```c
// adlist.c
list *listAddNodeTail(list *list, void *value)
```

同上，不过这个函数是将节点添加到列表尾部。



### listInsertNode

#### 函数原型

```c
// adlist.c
list *listInsertNode(list *list, listNode *old_node, void *value, int after)
```

这个函数用来想列表中指定节点前或后插入元素；

当 after 为 0 时，新节点在 old_node 前边；而为 0 时，则在后边；



### listDelNode

#### 函数原型

```c
// adlist.c
void listDelNode(list *list, listNode *node)
```

从指定列表中删除指定节点，并释放相关内存。



### listGetIterator

#### 函数原型

```c
// adlist.c
listIter *listGetIterator(list *list, int direction)
```

对一个列表生成一个迭代器，这个迭代器由 directiron 指定迭代方向，0 表示从表头开始，非 0 表示逆序。



### listReleaseIterator

#### 函数原型

```c
// adlist.c
void listReleaseIterator(listIter *iter)
```

释放整个迭代器结构体。



### listRewind listRewindTail

#### 函数原型

```c
// adlist.c
void listRewind(list *list, listIter *li) {
    li->next = list->head;
    li->direction = AL_START_HEAD;
}

void listRewindTail(list *list, listIter *li) {
    li->next = list->tail;
    li->direction = AL_START_TAIL;
}
```

这两个函数主要是在一个迭代器内部，强制性改变一个迭代器从旧列表指向新列表。



### listNext

#### 函数原型

```c
// adlist.c
listNode *listNext(listIter *iter)
```

从迭代器中返回下一个节点。



### listDup

#### 函数原型

```c
// adlist.c
list *listDup(list *orig)
{
    ...
        if (copy->dup) {
                value = copy->dup(node->value);
                if (value == NULL) {
                    listRelease(copy);
                    return NULL;
                }
            } else
                value = node->value;
    ...
}
```

基于一个列表复制出一个新的列表，它内部实现的过程中，用到了列表的 dup 函数指针。



### listSearchKey

#### 函数原型

```c
// adlist.c
listNode *listSearchKey(list *list, void *key)
```

在列表中查找是否有节点与给定的 key 相等，这个函数内部同样使用了列表的 match 函数指针。



### listIndex

#### 函数原型

```c
// adlist.c
listNode *listIndex(list *list, long index)
```

这个函数返回列表中指定索引的节点，从 0 开始计数，支持副索引；



### void listRotate(list *list)

#### 函数原型

```c
// adlist.c
void listRotate(list *list)
```

这个函数将一个列表中的最后一个元素插入到列表头，可能实际业务中有这种需求。