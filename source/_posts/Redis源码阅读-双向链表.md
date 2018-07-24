---
title: Redis源码阅读-双向链表
date: 2018-03-16 17:35:39
tags:
- 源码
- Redis
- 双向链表
categories:
- REDIS
---

## 双向链表
&emsp;&emsp;List在Redis中是一个比较常见的数据结构，实现的代码很简洁且高效，下面将逐步从代码级别对列表数据结构类型进行分析。
<!--more-->
### 数据结构定义

#### 单节点定义
```C
typedef struct listNode {
    struct listNode *prev;
    struct listNode *next;
    void *value;
} listNode;
```

单个节点包含三个元素：上一个节点、下一个节点、当前节点的值。所以可以看出Redis对列表类型结构采取双向链表数据结构实现的。

#### 列表定义
```C
typedef struct list {
    listNode *head;
    listNode *tail;
    void *(*dup)(void *ptr);
    void (*free)(void *ptr);
    int (*match)(void *ptr, void *key);
    unsigned long len;
} list;
```
通过下面一张图可以很好的表达这个结构：
{% asset_img list.png %}
下面解释一下结构中的每一个属性：
- head：指向第一个节点的元素指针
- tail：指向最后一个节点的元素指针
- dup：定义复制整个list的函数指针，可以用户自定义。
- free：释放单个元素的内存空间函数指针，可以用户自定义。
- match：匹配value的函数指针，可以用户自定义。
- len：当前列表共有多少个节点。

#### 迭代器定义
```C
typedef struct listIter {
    listNode *next;
    int direction;
} listIter;
```
Reids中为了更简单的对列表进行遍历，实现了一个代码简洁的迭代器。
- next：下一个元素的指针
- direction：遍历的方向。
 - AL_START_HEAD：表示从头开始遍历。
 - AL_START_TAIL：表示从尾部开始遍历。

稍后将分析如何通过这两个属性对整个列表进行迭代的。

### 链表操作
#### 创建链表
```C
list *listCreate(void)
{
    struct list *list;

    if ((list = zmalloc(sizeof(*list))) == NULL)
        return NULL;
    list->head = list->tail = NULL;
    list->len = 0;
    list->dup = NULL;
    list->free = NULL;
    list->match = NULL;
    return list;
}
```
链表创建是一个很简单的操作，创建后的内存状态如下图所示：
{% asset_img create.png %}

#### 插入元素
- 采取头插法插入一个元素：
```C
list *listAddNodeHead(list *list, void *value)
{
    listNode *node;
    //申请新节点的内存空间
    if ((node = zmalloc(sizeof(*node))) == NULL)
        return NULL;
    //设置值
    node->value = value;
    //如果当前列表长度为0，那就说明新加的节点即是头节点也是尾节点，故需要特殊处理
    if (list->len == 0) {
        list->head = list->tail = node;
        node->prev = node->next = NULL;
    } else {
        //将新节点插入到头部
        node->prev = NULL;
        node->next = list->head;
        list->head->prev = node;
        list->head = node;
    }
    list->len++;
    return list;
}
```
该函数将在列表的头部插入一个元素，也是命令`LPUSH`的实现。下面通过一个图，添加后的列表状态如下所示：
{% asset_img leftinsert.png %}

- 采取尾插法插入一个元素：
```C
list *listAddNodeTail(list *list, void *value)
{
    listNode *node;

    if ((node = zmalloc(sizeof(*node))) == NULL)
        return NULL;
    node->value = value;
    if (list->len == 0) {
        list->head = list->tail = node;
        node->prev = node->next = NULL;
    } else {
        node->prev = list->tail;
        node->next = NULL;
        list->tail->next = node;
        list->tail = node;
    }
    list->len++;
    return list;
}

```
该函数将在列表的尾部插入一个元素，也是命令`RPUSH`的实现。下面通过一个图，添加后的列表状态如下所示：
{% asset_img rightinsert.png %}

#### 元素查找

- 根据下标找到指定的元素：下标可以为负数，-1代表是最后一个元素。
```C
listNode *listIndex(list *list, long index) {
    listNode *n;
    if (index < 0) {
        index = (-index)-1;
        n = list->tail;
        while(index-- && n) n = n->prev;
    } else {
        n = list->head;
        while(index-- && n) n = n->next;
    }
    return n;
}
```
从上面的代码逻辑可以看出：下标的正负可以理解为方向，0和正数代表从左向右方向的下标，负数代表从右向左的下标，-1代表右边第一个，也就是通过这个机制在redis中可以使用负数下标。对于一个长度为4的列表，下标可以表示如下：
{% asset_img index.png %}

- 根据元素的值，查找list中的节点
```C
listNode *listSearchKey(list *list, void *key)
{
    listIter iter;
    listNode *node;

    listRewind(list, &iter);
    while((node = listNext(&iter)) != NULL) {
        if (list->match) {
            if (list->match(node->value, key)) {
                return node;
            }
        } else {
            if (key == node->value) {
                return node;
            }
        }
    }
    return NULL;
}
```
该函数会遍历list中的每一个元素，如果实现了list中的match函数，则会按照match函数的功能进行节点值的比对，否则将通过比较指针来判断是否是相同的value值。返回list中的节点类型`listNode`

#### 合并两个list
```C
void listJoin(list *l, list *o) {
    if (o->head)
        //将o的头部指向l的尾部
        o->head->prev = l->tail;

    if (l->tail)
        //将l的尾部next域指向o的head
        l->tail->next = o->head;
    else
        l->head = o->head;

    l->tail = o->tail;
    //合并长度
    l->len += o->len;
    o->head = o->tail = NULL;
    o->len = 0;
}

```
该函数的功能为：将名为`o`的list，添加到名为`l`的list的尾部。

#### 列表迭代器
- 获取一个迭代器：
```C
listIter *listGetIterator(list *list, int direction)
{
    listIter *iter;
    //申请一个迭代器内存
    if ((iter = zmalloc(sizeof(*iter))) == NULL) return NULL;
    //根据迭代的方向，确认第一个节点
    if (direction == AL_START_HEAD)
        iter->next = list->head;
    else
        iter->next = list->tail;
    iter->direction = direction;
    return iter;
}
```
一个迭代器一次只会持有一个节点，并且会根据`direction`变量进行初始化

- 获取迭代器的下一个值
```C
listNode *listNext(listIter *iter)
{
    listNode *current = iter->next;

    if (current != NULL) {
        if (iter->direction == AL_START_HEAD)
            iter->next = current->next;
        else
            iter->next = current->prev;
    }
    return current;
}
```
因为迭代器中持有了当前节点的指针，因为是双向链表，所以即可访问到前一个节点，也可以访问到下一个节点，具体下一个访问到哪个节点，取决于这个迭代器遍历的方向

- 迭代器状态的变更
```C
//将`list`列表的迭代器`li`状态重置为从头部开始迭代，且当前节点为头结点
void listRewind(list *list, listIter *li) {
    li->next = list->head;
    li->direction = AL_START_HEAD;
}
//将`list`列表的迭代器`li`状态重置为从尾部开始迭代，且当前节点为尾结点
void listRewindTail(list *list, listIter *li) {
    li->next = list->tail;
    li->direction = AL_START_TAIL;
}
```
