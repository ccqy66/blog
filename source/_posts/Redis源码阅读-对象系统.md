---
title: Redis源码阅读-对象系统
date: 2018-03-28 15:48:29
tags:
- 源码
- Redis
- 对象系统
categories:
- Redis
---
## 对象系统
&emsp;&emsp;前面讲到的数据结构是Redis底层存储数据使用的数据结构，但是Redis在操作时并不是直接对底层数据结构进行操作，而是通过操作封装的对象`robj`来间接的操作底层数据，屏蔽了不同数据结构不同操作。并且内置了对象的引用计数机制来进行内存回收。
<!--more-->
## 数据结构

### 结构定义
```C
typedef struct redisObject {
    /**
     * 类型，5种不同的类型
     * REDIS_STRING
     * REDIS_LIST
     * REDIS_SET
     * REDIS_HASH
     * REDIS_ZSET
     */
    unsigned type:4;
    /**
     * 编码，8中不同编码
     * REDIS_ENCODING_INT
     * REDIS_ENCODING_EMBSTR
     * REDIS_ENCODING_RAW
     * REDIS_ENCODING_HT
     * REDIS_ENCODING_LINKEDLIST
     * REDIS_ENCODING_ZIPLIST
     * REDIS_ENCODING_INTSET
     * REDIS_ENCODING_SKIPLIST
     */
    unsigned encoding:4;
    /**
     * 该对象最后一次访问的时间
     */
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits decreas time). */
    /**
     * 该对象的引用计数器
     */
    int refcount;
    /**
     * 指向实际值的指针
     */
    void *ptr;
} robj;
```
关于type和encoding的说明：
在Redis中数据类型有5种：字符串、列表、集合、哈希表、有序集合。这五种数据类型每种都可能有一种或者多种编码方式：

|类型|编码|对象|
|-|-|-|
|REDIS_STRING	|REDIS_ENCODING_INT|整数值实现的字符串对象|
||REDIS_ENCODING_EMBSTR|embstr编码实现的简单动态字符串实现的字符串对象|
||REDIS_ENCODING_RAW|简单动态字符串实现的字符串对象|
|REDIS_LIST|REDIS_ENCODING_ZIPLIST|压缩列表实现的列表对象|
||REDIS_ENCODING_LINKEDLIST|双端链表实现的列表对象|
|REDIS_HASH|REDIS_ENCODING_ZIPLIST|压缩列表实现的哈希对象|
||REDIS_ENCODING_HT|字典实现的哈希对象|
|REDIS_SET|REDIS_ENCODING_INTSET|整数集合实现的集合对象|
||REDIS_ENCODING_HT|字典实现的集合对象|
|REDIS_ZSET|REDIS_ENCODING_ZIPLIST|压缩列表实现的有序集合对象|
||REDIS_ENCODING_SKIPLIST|跳表实现的有序集合对象|

### 引用计数
&emsp;&emsp;C语言不像Java等面向对象的语言一样，支持垃圾回收，为了防止内存泄露，不会使用到的对象要计数清除。Redis处理垃圾回收使用的是引用计数算法，一般的原则是：当该对象被其他对象引入时，引用计数器加1，当其他对象释放对该对象的引用时，引用计数减1。

#### 引用计数增加
```C
void incrRefCount(robj *o) {
    if (o->refcount != OBJ_SHARED_REFCOUNT) o->refcount++;
}
```
当该对象被其他程序使用时，会调用`incrRefCount`函数，增加`o`对象的引用计数。

#### 引用计数释放
```C
void decrRefCount(robj *o) {
    if (o->refcount == 1) {
        switch(o->type) {
        case OBJ_STRING: freeStringObject(o); break;
        case OBJ_LIST: freeListObject(o); break;
        case OBJ_SET: freeSetObject(o); break;
        case OBJ_ZSET: freeZsetObject(o); break;
        case OBJ_HASH: freeHashObject(o); break;
        case OBJ_MODULE: freeModuleObject(o); break;
        default: serverPanic("Unknown object type"); break;
        }
        zfree(o);
    } else {
        if (o->refcount <= 0) serverPanic("decrRefCount against refcount <= 0");
        if (o->refcount != OBJ_SHARED_REFCOUNT) o->refcount--;
    }
}
```
当该对象不再被其他程序使用时，引用计数减1，当引用计数等于0时，释放该对象的内存空间。

### 共享对象
&emsp;&emsp;为了节约内存的目的，Redis引入了共享对象的概念，Redis通过将引用计数设置为INT_MAX(2147483647)的方式，将一个对象标志为一个共享对象。为什么说共享对象可以解决内存呢？
&emsp;&emsp;假设A、B两个程序都需要创建100这个int类型的对象，首先A创建了一个100的内存对象，之后将其标记为共享对象，如此在内存中的表示为：
{% asset_img shared.png %}
&emsp;&emsp;之后B程序也需要创建100这个对象，但是发现内存中已经存在了100这个共享对象，于是就不需要再重新创建，而是直接使用这个共享对象,于是这个数值为100的对象就被使用了两次，起到了节约内存的目的：
{% asset_img shared01.png %}
通过调用函数`makeObjectShared`将一个对象转为共享对象：
```C
robj *makeObjectShared(robj *o) {
    serverAssert(o->refcount == 1);
    o->refcount = OBJ_SHARED_REFCOUNT;
    return o;
}
```
## 不同对象
从上面的描述，我们知道，在Redis中有五种不同的对象类型：字符串、列表、哈希表、集合、有序集合。下面将分别分析这五种不同的对象体系。

#### 字符串对象
&emsp;&emsp;字符串可使用的编码为：REDIS_ENCODING_INT、REDIS_ENCODING_EMBSTR、REDIS_ENCODING_RAW。Redis将根据不同的字符串内容，采取不同的编码方式。
- REDIS_ENCODING_INT 编码：当字符串可以被转成long类型时，Redis将采取REDIS_ENCODING_INT编码
```C
robj *createStringObjectFromLongLong(long long value) {
    robj *o;
    //由于Redis默认创建了0~9999范围内的共享int对象，所以如果value在0~9999之间，就直接返回共享对象，避免重复创建对象
    if (value >= 0 && value < OBJ_SHARED_INTEGERS) {
        incrRefCount(shared.integers[value]);
        o = shared.integers[value];
    } else {
        //如果value在long范围内，就创建long类型的对象
        //类型为：OBJ_STRING
        //编码为：OBJ_ENCODING_INT
        if (value >= LONG_MIN && value <= LONG_MAX) {
            o = createObject(OBJ_STRING, NULL);
            o->encoding = OBJ_ENCODING_INT;
            o->ptr = (void*)((long)value);
        } else {
            //如果value的返回超过了long类型，就创建一个OBJ_ENCODING_RAW编码的字符串
            o = createObject(OBJ_STRING,sdsfromlonglong(value));
        }
    }
    return o;
}
```

  为了优化内存分类策略，Redis会根据字符串的长度采取两种不同的编码OBJ_ENCODING_EMBSTR和REDIS_ENCODING_RAW。当字符串长度小于44时，会采取OBJ_ENCODING_EMBSTR编码，否则采取REDIS_ENCODING_RAW编码。
```C
#define OBJ_ENCODING_EMBSTR_SIZE_LIMIT 44
robj *createStringObject(const char *ptr, size_t len) {
    if (len <= OBJ_ENCODING_EMBSTR_SIZE_LIMIT)
        return createEmbeddedStringObject(ptr,len);
    else
        return createRawStringObject(ptr,len);
}
```
- REDIS_ENCODING_EMBSTR 编码：当字符串长度小于44时会采取此编码，目的是为了优化内存分配。
```C
robj *createEmbeddedStringObject(const char *ptr, size_t len) {
    robj *o = zmalloc(sizeof(robj)+sizeof(struct sdshdr8)+len+1);
    struct sdshdr8 *sh = (void*)(o+1);

    o->type = OBJ_STRING;
    o->encoding = OBJ_ENCODING_EMBSTR;
    o->ptr = sh+1;
    o->refcount = 1;
    if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) {
        o->lru = (LFUGetTimeInMinutes()<<8) | LFU_INIT_VAL;
    } else {
        o->lru = LRU_CLOCK();
    }

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
```
  为什么长度限制在44范围内才使用这个编码呢？为什么可以起到优化内存分配呢？
首先看一下内存分配的代码：`robj *o = zmalloc(sizeof(robj)+sizeof(struct sdshdr8)+len+1);`
sizeof(robj)的内存大小为：4+4+24+32+32 = 96（12 byte）
sizeof(struct sdshdr8)的内存大小为：8+8+8+32 = 56（7 byte）
如果此时len=44 那么 此次申请的内存空间为：12+7+44+1 = 64(byte)
解释：
有了上述的计算，现在我们要了解一下jemalloc的内存分配规则：jemalloc一次所能分配的内存大小为：8、16、32、64。jemalloc一次最大申请64字节，所以只要len<=44，就可以满足一次内存申请就够用了，而不需要多次申请了。所以Redis就是通过这种方式来处理短字符串的处理。

- REDIS_ENCODING_RAW 编码：就是原生的字符串编码，没有任何的特殊处理：
```C
robj *createRawStringObject(const char *ptr, size_t len) {
    return createObject(OBJ_STRING, sdsnewlen(ptr,len));
}
```

- 编码转换

|之前编码|修改|修改后编码|
|-|-|-|
|REDIS_ENCODING_INT|数字->非数字|REDIS_ENCODING_RAW|
|REDIS_ENCODING_INT|数字->数字|REDIS_ENCODING_INT|
|REDIS_ENCODING_RAW|修改|REDIS_ENCODING_RAW|
|REDIS_ENCODING_EMBSTR|修改|REDIS_ENCODING_RAW|
