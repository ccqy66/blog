---
title: Redis源码阅读-整数集合
date: 2018-03-27 20:27:15
tags:
- 源码
- Redis
- 整数集合
categories:
- Redis
---
## 整数集合
&emsp;&emsp;整数集合是集合的底层实现，当一个集合中只包含整数元素时，并且该集合中的元素不多时，采取整数集合进行存储。
<!--more-->
## 数据结构定义
```C
typedef struct intset {
    uint32_t encoding;
    uint32_t length;
    int8_t contents[];
} intset;
```
该数据结构只包含3个属性：
- encoding：标识着当前元素采取多少位存储。包含3个可选的值：INTSET_ENC_INT16（2）、INTSET_ENC_INT32（4）、INTSET_ENC_INT64（8）。
- length：集合中的数据元素个数。并不是数组的长度，如果当前的encoding为INTSET_ENC_INT32，那就说明需要用4个字节来保存一个值，也就需要4个数组长度来保存一个值，假设当前数组的长度为16，也只能说明length=4（16/4）。
- contents：底层存储数据的数组。一个多位表示的数字，需要多个元素来存储。例如：对于一个64位 8字节的数字，需要有8个元素来保存一个值

假设当前集合中有3个值，分别为：10、47、310，那么对应于内存中的存储如下图所示：
{% asset_img 01.png %}
初始的编码是使用INTSET_ENC_INT16（2)，所以一个元素使用2个字节来存储，也就是说使用2个数组长度来保存一个值。
如果此时再新增加一个元素，该值通过两个字节无法表示了，就需要进行升级。例如当前新增加一个值为750000的元素，因为当前编码所能表示的最大值为-32768~32767，所以就需要升级到INTSET_ENC_INT32（4），使用4个字节表示一个值。升级后的内存中的存储如下图所示：
{% asset_img 02.png %}

升级后有如下需要注意的点：
- 原有的元素也要使用新的编码进行存储。例如上图中的10也需要使用4个字节进行存储，虽然一个字节就已经足够了。

## 整数集合操作：
### 单字节数组存储
从上述的描述我们知道：一个多字节的元素是通过一个个单字节的数组元素来承载的，具体到Redis是采取什么样的操作来完成的呢？也就是说：Redis是如何将一个多字节的元素保存到这个单字节的数组中，又是如何从单字节的数组中解码出这个多字节的元素的呢？

#### 多字节元素编码
直接上代码：
```C
static void _intsetSet(intset *is, int pos, int64_t value) {
    uint32_t encoding = intrev32ifbe(is->encoding);

    if (encoding == INTSET_ENC_INT64) {
        ((int64_t*)is->contents)[pos] = value;
        memrev64ifbe(((int64_t*)is->contents)+pos);
    } else if (encoding == INTSET_ENC_INT32) {
        ((int32_t*)is->contents)[pos] = value;
        memrev32ifbe(((int32_t*)is->contents)+pos);
    } else {
        ((int16_t*)is->contents)[pos] = value;
        memrev16ifbe(((int16_t*)is->contents)+pos);
    }
}
```
编码逻辑很简单，首先判断当前的编码类型，然后将数组指针强转成指定类型的指针，直接写入即可。例如：如果当前编码是INTSET_ENC_INT64类型，那么就会在指定下标下写入4字节的内容。从而达到将一个多字节的数字写入到多个连续的单字节数组中。
#### 多字节元素解码
```C
static int64_t _intsetGetEncoded(intset *is, int pos, uint8_t enc) {
    int64_t v64;
    int32_t v32;
    int16_t v16;

    if (enc == INTSET_ENC_INT64) {
        memcpy(&v64,((int64_t*)is->contents)+pos,sizeof(v64));
        memrev64ifbe(&v64);
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
解码过程主要是使用了`memcpy`函数，该函数可以完成从指定的内存起始位置开始，拷贝n个字节。从而达到从单字节的数组中读取指定字节的数据。

### 创建整数集合
```C
intset *intsetNew(void) {
    intset *is = zmalloc(sizeof(intset));
    /**
     * 默认采取2个字节
     */
    is->encoding = intrev32ifbe(INTSET_ENC_INT16);
    is->length = 0;
    return is;
}
```
这个函数简单，没有什么可以说的，需要注意一点，默认情况下，使用2个字节来保存单个元素

### 插入
```C
intset *intsetAdd(intset *is, int64_t value, uint8_t *success) {
    /**
     * 根据value值，判断当前value的类型。
     * 计算出来的是当前value是使用几个字节保存的
     */
    uint8_t valenc = _intsetValueEncoding(value);
    uint32_t pos;
    if (success) *success = 1;

    /* Upgrade encoding if necessary. If we need to upgrade, we know that
     * this value should be either appended (if > 0) or prepended (if < 0),
     * because it lies outside the range of existing values. */
    /**
     * 如果当前value的编码类型大于集合当前的编码类型，就需要进行升级
     */
    if (valenc > intrev32ifbe(is->encoding)) {
        /* This always succeeds, so we don't need to curry *success. */
        return intsetUpgradeAndAdd(is,value);
    } else {
        /* Abort if the value is already present in the set.
         * This call will populate "pos" with the right position to insert
         * the value when it cannot be found. */
        if (intsetSearch(is,value,&pos)) {
            if (success) *success = 0;
            return is;
        }
        //增加一个内存块保存数据：使用realloc直接申请一个新的内存块
        is = intsetResize(is,intrev32ifbe(is->length)+1);
        //将pos之后的元素全部右移一位
        if (pos < intrev32ifbe(is->length)) intsetMoveTail(is,pos,pos+1);
    }
    //将该值插入到pos位置
    _intsetSet(is,pos,value);
    //重置length
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
    return is;
}
```
插入一个新的元素的流程如下：
- 首先计算要插入元素的编码（也就说需要插入的元素最少使用什么类型的编码才可以存储）
- 如果要插入元素的编码大于集合当前编码，就需要进行集合升级。
- 否则，先查询集合中是否存在该元素，如果存在就不再插入，直接返回。
- 如果不存在，扩充集合的长度,然后将新值插入到指定的位置。

### 升级
如果新插入的值大于当前集合的编码，那就需要进行集合编码的升级。
```C
static intset *intsetUpgradeAndAdd(intset *is, int64_t value) {
    //当前集合的编码
    uint8_t curenc = intrev32ifbe(is->encoding);
    //插入值的编码
    uint8_t newenc = _intsetValueEncoding(value);
    //当前集合元素个数
    int length = intrev32ifbe(is->length);
    int prepend = value < 0 ? 1 : 0;

    /* First set new encoding and resize */
    //重置当前集合的编码
    is->encoding = intrev32ifbe(newenc);
    //扩充当前集合的长度
    is = intsetResize(is,intrev32ifbe(is->length)+1);

    /* Upgrade back-to-front so we don't overwrite values.
     * Note that the "prepend" variable is used to make sure we have an empty
     * space at either the beginning or the end of the intset. */
    //从后向前，对每个元素进行升级
    while(length--)
        _intsetSet(is,length+prepend,_intsetGetEncoded(is,length,curenc));

    /* Set the value at the beginning or the end. */
    //将新值插入集合头部或者尾部
    if (prepend)
        _intsetSet(is,0,value);
    else
        _intsetSet(is,intrev32ifbe(is->length),value);
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
    return is;
}
```
