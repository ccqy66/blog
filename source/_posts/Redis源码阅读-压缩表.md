---
title: Redis源码阅读-压缩表
date: 2018-04-02 20:59:06
tags:
- 源码
- Redis
- 双向链表
categories:
- Redis
---
## 压缩表
&emsp;&emsp;压缩列表是列表和哈希的底层实现之一。当一个列表只包含少量列表项时，并且每个列表项要么就是小整数值，要么就是长度比较短的字符串，那么Redis就会使用压缩列表来做列表的底层实现。--《Redis设计与实现》
<!--more-->
### 为什么需要压缩表
&emsp;&emsp;在研究Redis的压缩表时，有一个问题困扰着我，已经有了双端链表了，为什么还需要压缩表呢？官方的说法是为了减少内存的损耗，体现在哪里呢？
为了解决这个问题，首先我们把二者的内存结构展示出来：
双端链表定义：
```C
typedef struct listNode {
    struct listNode *prev;
    struct listNode *next;
    void *value;
} listNode;
typedef struct list {
    listNode *head;
    listNode *tail;
    void *(*dup)(void *ptr);
    void (*free)(void *ptr);
    int (*match)(void *ptr, void *key);
    unsigned long len;
} list;
```
从上面的定义来看，一个`listNode`占用的内存为24字节。
压缩表格式：
```C
<zlbytes><zltail><zllen><entry>...<entry><zlend>
```
节点格式：
```c
|<prelen><<encoding+len><len>><data>|
|---1----------------2--------------3---|
<prelen>：占用1个字节或者5个字节
<encoding+len><len>：占用1个字节或者5个字节
<data>：保存的为数据的地址，占用4个字节
```
从上面的分析来看，一个entry最少占用6（1+1+4）个字节，最大占用14（5+5+4）个字节。所以entry占用的字节数在`6~14`之间。
这样的话，使用压缩表最多可以省（24->6）75%，最少可以省（24->14）41.7%，故可以节省的内存在`41.7%~75%`之间。
结论：所以从上面的分析来看，使用压缩表的确可以大大的起到节约内存的作用。

### 结构分析
&emsp;&emsp;我们已经知道了，压缩表可以起到节约内存的作用，现在我们从底层分析，为什么这样可以压缩内存？这样做是否用影响到数据存取过程中的性能呢？
#### 压缩表结构
```C
<zlbytes><zltail><zllen><entry>...<entry><zlend>
```
- zlbytes：占用4个字节，使用无符号的整数表示，采取小端存储，表示当前压缩表占用的字节数，包括zlbytes本身占用的内存空间。
- zltail：占用4个字节，记录了当前压缩表的最后一个元素的偏移量。
- zllen：占用2个字节，记录了当前压缩表的元素个数。当该值大于2^16-1时，就不能通过该值获取当前压缩表元素的个数了，需要进行遍历了。
- zlend：占用1个字节，是一个常量数255，标识着压缩表的结束。

#### 节点结构
```C
<prevlen><encoding+lensize><len><entry-data>
```
- prevlen：保存前一个元素占用的内存字节数，如果前一个元素占用的内存字节数小于255bytes，就使用1字节保存，否则使用5字节保存，第一个字节保存:FE，表明使用5字节表示前一个元素占用的字节数，后4个字节真正表示前一个元素占用的字节数。
下图是表示长度为：8239和47 两种情况对应的内存表示：
{% asset_img 8239.png %}
- encoding+lensize ：表示当前节点内容的编码以及保存内容所占用的字节数。压缩表保存两种数据类型：字符串和整数
  - 字符串编码：前两位数据编码位。
   - ZIP_STR_06B（00 000000）：00-表示该节点使用1个字节表示编码和长度，后6位用来保存字符串长度。
   - ZIP_STR_14B（01 000000）：01-表示该节点使用2个字节表示编码和长度，后14位用来保存字符串长度。
   - ZIP_STR_32B（10 000000）：10-表示该节点使用5个字节表示编码和长度，第1个字节表示编码，后4个字节表示字符串的长度。
  - 数字编码：使用1个字节来同时表示编码类型和内容长度。
   - ZIP_INT_8B（11111110）：表示使用1个字节来表示数字。
   - ZIP_INT_16B（11000000）：表示使用2个字节表示编码。
   - ZIP_INT_24B（11110000）：表示使用3个字节表示编码。
   - ZIP_INT_32B（11010000）：表示使用4个字节表示编码。
   - ZIP_INT_64B（11100000）：表示使用8个字节表示编码。
- len：表示当前节点的长度，使用0个字节、1个字节或者4个字节保存
  - 0个字节：
   - 当编码为ZIP_STR_06B时，后6个位就表示长度，就不需要单独的len字段来保存了。
   - 当编码为数字时，所有的数字编码都不需要单独的len字段来保存。
  - 1个字节：
   - 当编码为ZIP_STR_14B时，使用编码字段的后6位+len字段的8位，使用14位来表示长度
  - 4个字节
   - 当编码为ZIP_STR_32B时，len字段使用4字节来表示长度。
- data：表示指向具体数据的指针，占用4个字节。
#### 举例
上面的描述有点繁琐，现在我们举个例子来说明一下。
假设我们存储3个元素，分别为：字符串（"hello world"），数字（123），字符串（"my name is chenchen"）。
先后执行lpush命令，并将压缩表的内存信息打印到文件，执行`od -cx dump`，得到如下信息：
```C
0000000    0  \0  \0  \0   "  \0  \0  \0 003  \0  \0 023   m   y       n
          060 000 000 000 042 000 000 000 003 000 000 023 155 171 040 156
0000020    a   m   e       i   s       c   h   e   n   c   h   e   n 025
          141 155 145 040 151 163 040 143 150 145 156 143 150 145 156 025
0000040  376   { 003  \v   h   e   l   l   o       w   o   r   l   d 377
          376 173 003 013 150 145 154 154 157 040 167 157 162 154 144 377

```
现在我们按照之前所了解的内存布局来一个字节一个字节的分析这些信息（小端存储）：
压缩表头部信息：
- 0~4字节：前4个字节表示当前压缩表总共占用多少字节：060 000 000 000（48），表示当前压缩表总共占用48的字节。
- 5~8字节：该区域表示压缩表中最后一个元素的偏移量：042 000 000 000（34），表示最后一个元素相对于首地址的偏移量为34。
- 9~10字节：该区域表示压缩表中一共有多少个元素：003 000（3），表示当前压缩表有3个元素。
压缩表元素信息：
第一个元素：
- 11 字节：当前元素的前一个节点的内存大小：000（0），表示前一个元素占用的字节数为0，也同时说明是第一个元素。
- 12 字节：当前元素的编码以及长度：023（19），表示当前元素为字符串，且长度为19。
- 13~31字节：当前当前元素的内容为："my name is chenchen"。
第二个元素：
- 32字节：当前元素的前一个节点的内存大小为：025（21），表示前一个元素占用的字节数为21字节。
- 33字节：当前元素的编码以及长度：376（254），表示当前元素为数字，且数字使用1字节编码。
- 34字节：当前当前元素的内容为：173（123），表示当前元素的内容为123。
第三个元素：
- 35字节：当前元素的前一个节点的内存大小为：003（3），表示前一个元素占用的字节数为3字节。
- 36字节：当前元素的编码以及长度：013（11），示当前元素为字符串，且长度为11。
- 37~47字节：当前当前元素的内容为："hello world"。
- 48字节：内容为：377（255），表示当前压缩表结束。

### 压缩表操作
#### 创建一个新的压缩表
```c
unsigned char *ziplistNew(void) {
    //|zlbytes|zltail|zllen|zlend|
    //|8|8|2|1|
    //2*8+2+1 = 19(bytes)
    unsigned int bytes = ZIPLIST_HEADER_SIZE+1;
    unsigned char *zl = zmalloc(bytes);
    //将指针转成32位存储，并使用小端存储
    ZIPLIST_BYTES(zl) = intrev32ifbe(bytes);
    ZIPLIST_TAIL_OFFSET(zl) = intrev32ifbe(ZIPLIST_HEADER_SIZE);
    ZIPLIST_LENGTH(zl) = 0;
    //最后一位为结束符：255[FF]
    zl[bytes-1] = ZIP_END;
    return zl;
}
```
创建一个压缩很简单，主要是初始化zlbytes、zltail、zllen、zlend4块内容。

#### 添加元素
在该函数中，逻辑稍微有点复杂。
```c
unsigned char *__ziplistInsert(unsigned char *zl, unsigned char *p, unsigned char *s, unsigned int slen) {
  size_t curlen = intrev32ifbe(ZIPLIST_BYTES(zl)), reqlen;
  /**
   * prevlensize：前一个节点长度的位数
   * prevlen：前一个节点的长度
   */
  unsigned int prevlensize, prevlen = 0;
  size_t offset;
  int nextdiff = 0;
  unsigned char encoding = 0;
  long long value = 123456789; /* initialized to avoid warning. Using a value
                                  that is easy to see if for some reason
                                  we use it uninitialized. */
  zlentry tail;

  /* Find out prevlen for the entry that is inserted. */
  //查询出被插入的那个节点的前一个节点的prevlen
  //如果是lpush：p指针指向第一个元素的首地址
  //如果是rpush：p指针指向结束符的首地址，即 FF

  //从头插入
  if (p[0] != ZIP_END) {
      // 获取当前节点的前一个节点的长度
      ZIP_DECODE_PREVLEN(p, prevlensize, prevlen);
  } else {
      //从尾部插入
      unsigned char *ptail = ZIPLIST_ENTRY_TAIL(zl);
      if (ptail[0] != ZIP_END) {
          prevlen = zipRawEntryLength(ptail);
      }
  }

  /* See if the entry can be encoded */
  // 尝试将字符串编码成整数
  if (zipTryEncoding(s,slen,&value,&encoding)) {
      /* 'encoding' is set to the appropriate integer encoding */
      reqlen = zipIntSize(encoding);
  } else {
      /* 'encoding' is untouched, however zipStoreEntryEncoding will use the
       * string length to figure out how to encode it. */
      reqlen = slen;
  }
  /* We need space for both the length of the previous entry and
   * the length of the payload. */
  // 保存前一个节点长度所需要的内存空间
  reqlen += zipStorePrevEntryLength(NULL,prevlen);
  // 保存当前节点编码所需要的内存空间
  reqlen += zipStoreEntryEncoding(NULL,encoding,slen);

  /* When the insert position is not equal to the tail, we need to
   * make sure that the next entry can hold this entry's length in
   * its prevlen field. */
  //当插入的位置不是尾部时，要确保下一个节点能够保持当前节点的长度
  //当前节点内容的长度下一个节点无法保存时，需要为下一个节点扩容，使其可以保存当前节点的长度
  int forcelarge = 0;
  nextdiff = (p[0] != ZIP_END) ? zipPrevLenByteDiff(p,reqlen) : 0;
  if (nextdiff == -4 && reqlen < 4) {
      nextdiff = 0;
      forcelarge = 1;
  }

  /* Store offset because a realloc may change the address of zl. */
  // 保存p 与 zl的偏移量，因为realloc 会改变zl的地址
  offset = p-zl;
  //扩容当前的数组
  zl = ziplistResize(zl,curlen+reqlen+nextdiff);
  //计算新数组的当前指针位置。
  p = zl+offset;

  /* Apply memory move when necessary and update tail offset. */
  if (p[0] != ZIP_END) {
      /* Subtract one because of the ZIP_END bytes */
      /**
       * 原数组：
       * |zlbytes|zltail|zllen|entry1|entry2|...|entryN|zlend|
       *       |
       * 新数组：
       * |zlbytes|zltail|zllen|entry1|entry2|...|entryN|zlend|new|
       *
       * 将p-nextdiff 之后的内容全部移到新数组的 p+reqlen之后
       * nextdiff：如果下一个节点无法保存当前节点的长度（1个字节），就需要扩容到5字节，而nextdiff就是这个5-1=4bytes
       */
      memmove(p+reqlen,p-nextdiff,curlen-offset-1+nextdiff);

      /* Encode this entry's raw length in the next entry. */
      //重置下一个节点的prev数据
      //如果下一个节点无法保存prev数据的话，就需要扩容
      if (forcelarge)
          zipStorePrevEntryLengthLarge(p+reqlen,reqlen);
      else
          zipStorePrevEntryLength(p+reqlen,reqlen);

      /* Update offset for tail */
      ZIPLIST_TAIL_OFFSET(zl) =
          intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+reqlen);

      /* When the tail contains more than one entry, we need to take
       * "nextdiff" in account as well. Otherwise, a change in the
       * size of prevlen doesn't have an effect on the *tail* offset. */
      zipEntry(p+reqlen, &tail);
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
  p += zipStorePrevEntryLength(p,prevlen);
  p += zipStoreEntryEncoding(p,encoding,slen);
  if (ZIP_IS_STR(encoding)) {
      memcpy(p,s,slen);
  } else {
      zipSaveInteger(p,value,encoding);
  }
  ZIPLIST_INCR_LENGTH(zl,1);
  return zl;
}

```
总结一下流程：
- 1、当执行lpush命令时，p指针指向第一个元素的首地址。当执行rpush命令时，p指针指向结束符的首地址，即：FF
- 2、首先尝试将value转成整数，如果转换成功，同时会得到`encoding`、和整数`value`
- 3、计算当前value需要占用的内存空间：reqlen = prevlen(上一个元素占用内存空间)+encoding(编码以及当前元素的长度所占用的内存空间)+slen(value占用的内存空间)
- 4、如果是从头结点插入新元素，需要确认当前头结点的元素prevlen字段是否可以保存当前新加元素的长度。
- 5、重新申请内存空间，保证可以添加两部分内容：当前节点占用内存+如果当前首节点prevlen字段不能保存长度需要扩容4个字节
- 6、将之前的数据重新分配到新申请的内存区块内，有两种拷贝可能，如下图：

下一节点的prevlen可以保存当前新加入节点的长度：
{% asset_img enough.png %}

下一节点的prevlen无法保存当前新加入节点的长度，需要扩容：
{% asset_img not_enough.png %}

- 7、将数据填充到新申请的内存区块内，并重置当前压缩表的状态数据。

#### 其他
&emsp;&emsp;通过前几章对压缩表底层原理的分析，加上对`insert`函数的分析，其他的逻辑再分析将不在话下，就不再此处赘述。
