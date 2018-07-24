---
title: Redis源码阅读-字典结构
date: 2018-03-18 17:47:51
tags:
- 源码
- Redis
- 字典
categories:
- REDIS
---
## 字典
&emsp;&emsp;Redis中的字典类似于Java语言中的`HashMap`，Redis的底层实现采取哈希表，字典在Redis中使用广泛，很多的结构都是采取字典来实现的，包括数据库的kv结构，下面我们将分析字典在Redis中的底层实现原理。解开其神秘面纱。
<!--more-->
### 数据结构定义

#### 单个元素定义
```C
typedef struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    //指向下一个节点，用来解决键的hash冲突，故采取拉链法解决hash冲突。
    struct dictEntry *next;
} dictEntry;
```
单个节点采取`key-value`的结构进行保存的，那必然就需要这么一个内存结构定义，如下图所示：
{% asset_img entry.png %}

#### 哈希表定义
```C
typedef struct dictht {
    //哈希表数组，每个元素都是一个指向dictEntry结构的指针
    dictEntry **table;
    //记录hash表的数量，也即是table数组的大小。
    unsigned long size;
    //用来计算size的掩码，总是等于size-1，防止溢出
    unsigned long sizemask;
    //记录该哈希表已有的节点数量
    unsigned long used;
} dictht;
```
内存结构定义如下图所示：
{% asset_img hashtable.png %}

#### 字典定义
```C
typedef struct dict {
    //类型特定函数
    dictType *type;
    //私有数据
    void *privdata;
    //哈希表，有两个，其中一个是用来保存真实哈希表的，另一个是在reash时使用
    dictht ht[2];
    //rehash索引，当rehash不在进行时，值为-1，记录了reash目前的进度
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    //
    unsigned long iterators; /* number of iterators currently running */
} dict;
```
内存结构定义如下图所示：
{% asset_img dict.png %}

#### 字典中的相关操作函数集合
```C
typedef struct dictType {
    //计算哈希值的函数
    uint64_t (*hashFunction)(const void *key);
    //赋值键的函数
    void *(*keyDup)(void *privdata, const void *key);
    //赋值值的函数
    void *(*valDup)(void *privdata, const void *obj);
    //键比较的函数
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);
    //销毁键的函数
    void (*keyDestructor)(void *privdata, void *key);
    //销毁值的函数
    void (*valDestructor)(void *privdata, void *obj);
} dictType;
```
该结构体中定义了对字典的操作函数，具体的实现会由数据类型的不同而不同。
### 字典操作

#### 哈希算法
&emsp;&emsp;既然Redis的字典实现是通过哈希表来完成的，那必然不能少了哈希算法，由于对字典中的操作是由一个`dictType`类型的结构体来实现的。所以不同的类型hash算法也有所不同，在4.0.2版本中，Redis使用的是`siphash`哈希算法来实现的。

#### Rehash
##### 为什么要
随着hash表的数据越来越多，为了尽大可能的减低hash冲突，并提高查找效率，那么就需要进行扩容，由于扩容后的hash表大小与之前大小不同，所以就需要进行重hash，将之前老表中的数据元素重新hash到新表中。

- Redis中如何触发rehash？
代码：`dict.c/_dictExpandIfNeeded`
```C
static int _dictExpandIfNeeded(dict *d)
{
    /**
     * 如果此时已经在rehash，就直接返回。同一时间只能有一次rehash过程。
     */
    /* Incremental rehashing already in progress. Return. */
    if (dictIsRehashing(d)) return DICT_OK;
    /**
     * 如果hash表当前为空，那么初始化hash表。
     */
    /* If the hash table is empty expand it to the initial size. */
    if (d->ht[0].size == 0) return dictExpand(d, DICT_HT_INITIAL_SIZE);
    /**
     * 满足以下条件需要进行扩容容量：
     * 1、hashtable数组全部被使用
     * 2、dict_can_resize=1
     * 3、或者装载因子>5
     */
    /* If we reached the 1:1 ratio, and we are allowed to resize the hash
     * table (global setting) or we should avoid it but the ratio between
     * elements/buckets is over the "safe" threshold, we resize doubling
     * the number of buckets. */
    if (d->ht[0].used >= d->ht[0].size &&
        (dict_can_resize ||
         d->ht[0].used/d->ht[0].size > dict_force_resize_ratio))
    {
        //将ht[1]的数组扩充为ht[0].used*2，并置d->rehashidx=0，表明此时处于rehash状态，且当前从ht[0].table[0]开始进行迁移
        return dictExpand(d, d->ht[0].used*2);
    }
    return DICT_OK;
}
```
当ht[0]表满足了上述所说的情况时，就需要rehash，但是在此步中并不会执行rehash的操作，只是完成了如下两件事：
1、初始化ht[1]数组大小为ht[0].used的2倍。
2、置当前字典的rehashidx=0，表明进入rehash状态。
而正在的rehash过程是分布在多次请求中完成了，下面将详细分析。

##### 如果进行
代码：`dict.c/dictRehash`
```C
int dictRehash(dict *d, int n) {
    int empty_visits = n*10; /* Max number of empty buckets to visit. */
    //只处理rehash过程
    if (!dictIsRehashing(d)) return 0;
    /**
     * 退出条件：
     * 1、访问到了empty_visits次的空节点。
     * 2、h[0]中所有的节点都被迁移完了
     * 3、完成了n步的迁移:迁移n个slot的数据
     */
    while(n-- && d->ht[0].used != 0) {
        dictEntry *de, *nextde;

        /* Note that rehashidx can't overflow as we are sure there are more
         * elements because ht[0].used != 0 */
        assert(d->ht[0].size > (unsigned long)d->rehashidx);
        //找到下一个不为null的节点，最多访问到empty_visits次的空节点就结束了本次的rehash。
        while(d->ht[0].table[d->rehashidx] == NULL) {
            d->rehashidx++;
            if (--empty_visits == 0) return 1;
        }
        //不为null的一个桶的节点
        de = d->ht[0].table[d->rehashidx];
        /**
         * 把该桶下的所有节点都移动到h[1]中。
         */
        /* Move all the keys in this bucket from the old to the new hash HT */
        while(de) {
            unsigned int h;

            nextde = de->next;
            /* Get the index in the new hash table */
            //计算新的key的hash值，实际上key的hash值的固定的，值的改变是由sizemask来确定的
            h = dictHashKey(d, de->key) & d->ht[1].sizemask;
            de->next = d->ht[1].table[h];
            d->ht[1].table[h] = de;
            d->ht[0].used--;
            d->ht[1].used++;
            de = nextde;
        }
        //置已经迁移的节点为NULL
        d->ht[0].table[d->rehashidx] = NULL;
        //rehashidx下标加一
        d->rehashidx++;
    }

    /* Check if we already rehashed the whole table... */
    //检查一个是不是rehash完成，当完成是d->ht[0].used==0
    if (d->ht[0].used == 0) {
        //释放0号为的内存空间
        zfree(d->ht[0].table);
        //将1号位赋值给0号位，保证了每次只使用0号位保存值，1号位用来rehash
        d->ht[0] = d->ht[1];
        _dictReset(&d->ht[1]);
        d->rehashidx = -1;
        return 0;
    }
    /* More to rehash... */
    return 1;
}
```
真实的rehash操作从上面的代码中可以看出，还是很简单的。首先说明一下该函数中比较重要的变量：`empty_visits`
> 该字段表示：对于本次rehash过程中，最多访问多少次空节点就结束了，防止大量访问空节点而导致大量的耗时。

下面我们简单描述一下`一次`rehash的过程（因为rehash分为多步进行的，一次仅会迁移一个桶的数据）：
1、首先判断一下，当前是否处于rehash状态，该函数只会在rehash状态时才会执行。
2、按照`rehashidx`下标遍历，直到找到一个不为NULL的桶为止，如果遍历了超过`empty_visits`次都没有找到，那么本次rehash就算是结束了。
3、把该桶下的所有节点全部rehash到新的hash表中（ht[1]表）。
4、最后判断一下是否rehash结束，如果结束了，就把h[0]=ht[1]，然后释放ht[1]的空间。

##### N步渐进式
如果数据库的数据量不是很大的情况下，执行rehash操作并没有什么影响，但是如果一个数据库中的数据非常多，又由于Redis处理请求是单线程的，所以会存在rehash过程会阻塞很多的请求。为了解决这个问题，将rehash过程分成多步执行，将一个耗时操作平均分配到一次次的请求中。并且每次rehash只会迁移一个桶的元素。
```C
。。。
if (dictIsRehashing(d)) _dictRehashStep(d);
。。。
static void _dictRehashStep(dict *d) {
    if (d->iterators == 0) dictRehash(d,1);
}
```
每次操作会有一次判断，当前是否处于rehash状态，如果是，就会执行一步rehash。

##### 字典遍历
对于一个正常的字典，进行遍历，并没有什么难度，但是Redis的字典是会进行N步渐进式rehash的，元素的位置会发生改变，对其进行正确的遍历还是一个很有难度的事情，但是Redis实现了，并且很高效，本来想写一篇介绍这个算法，但是在google时找到一篇，写的很好，可以直接过去看看：http://blog.csdn.net/gqtcgq/article/details/50533336
