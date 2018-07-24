---
title: Redis源码阅读-跳跃表
date: 2018-03-22 21:11:03
tags:
- 源码
- Redis
- 双向链表
categories:
- Redis
---

## 跳跃表
&emsp;&emsp;跳跃链表是一种随机化数据结构，基于并联的链表，其效率可比拟于二叉查找树(对于大多数操作需要O(log n)平均时间)，并且对并发算法友好。
<!--more-->
- 跳跃表的优势：
跳跃表在Redis中是实现有序集合的底层数据结构，有序集合在Redis是这么定义的：
 - 1）在集合内的元素不能重复。
 - 2）每个元素都关联一个分数，且有序体现在按照这个分数进行排序。

&emsp;&emsp;现在有两种简单的数据结构可以实现上述的要求，一是通过有序数组，二是通过有序链表实现。但是这两种方式都会有或多或少的有一些缺陷。
- 对于数组而言：向数组中插入元素的时间复杂度为`O(n)`，数组虽然可以通过二分查找（`O(lg(n))`）快速找到新元素插入的位置，但是插入后需要移动新元素之后的元素（`O(n)`）。
- 对于链表而言：虽然插入元素的时间复杂度是`O(1)`，但是查找插入位置的时间复杂度却是`O(n)`，总体上还是`O(n)`。

对于一个数据量大的有序集合而言，插入和删除的时间复杂度为`O(n)`是不可容忍的，于是就需要一个新的数据结构，可以满足插入和删除具有很高的效率。而跳跃表就是这么一个高效的算法，插入和删除的时间复杂度优化到`O(lg(n))`

&emsp;&emsp;从上述的描述我们知道，数组的优点是查找快，而链表的优点则是插入快，于是优化的思路可以是将二者的优点集为一体。直接给出结构图：
{% asset_img skiplist.png %}
&emsp;&emsp;从整体结构上看，结构的构造形式是以链表，但是不同的，每个节点有随机数个next指针，指向后多个节点。（除了首节点，首节点包含最大数量的next指针，在Redis中这个数字是32）。
&emsp;&emsp;为什么需要这样呢？我来举个例子，假设有如下所示的一个有序数列：1，3，4，6，7，9，10，15，21，34。使用双链表保存：
{% asset_img list_01.png %}
&emsp;&emsp;我们知道，在一个链表中查找一个元素的时间复杂度是0(n)，为了能够快速的查找元素，下面为这些元素提取一些'索引'，如下图所示：
{% asset_img list_02.png %}
此时如果要查找链表中的元素就不需要在从链表头部开始，一个一个的遍历的。举个例子，查询7这个值，首先遍历索引，查到了7肯定在6和15之间，那么此时直接从6开始往后遍历，下一个节点就是目标节点了。
&emsp;&emsp;为了更快，我们还可以为‘索引’创建下一层‘索引’，如下图所示：
{% asset_img list_03.png %}
这样，查询路径就变成了，先查最高层的索引，我们知道，目标节点在1和15之间。再查第二层索引，得知目标节点在6和15之间。最后查数据层，只需要遍历6和15之间的节点即可。最终只需要查询3次即可得到最终结果。

## 数据结构定义
### 跳跃表节点定义
```C
typedef struct zskiplistNode {
    sds ele;
    double score;
    //指向前一个节点
    struct zskiplistNode *backward;
    struct zskiplistLevel {
        //指向后续节点
        struct zskiplistNode *forward;
        //跨度，记录两个节点之间的距离
        unsigned int span;
    } level[];
} zskiplistNode;
```
节点定义中的属性含义：
- ele：跳跃节点中保存的元素信息。
- score：该节点的分数。
- backward：该节点的前一个节点。
- level：n个后续节点

后续节点描述属性含义：
- forward：后续节点同层的节点。
- span：跨度，记录了相邻两个节点之间的距离，该距离用来定义排名。该节点距离首节点的跨度越大，说明也就越靠后，排名也就越大。

### 跳跃表定义
```C
typedef struct zskiplist {
    /**
     * header ：指向跳跃表的表头节点。
     * tail ：指向跳跃表的表尾节点。
     * level ：记录目前跳跃表内，层数最大的那个节点的层数（表头节点的层数不计算在内）。
     * length ：记录跳跃表的长度，也即是，跳跃表目前包含节点的数量（表头节点不计算在内）。
     */
    struct zskiplistNode *header, *tail;
    unsigned long length;
    int level;
} zskiplist;
```
跳跃表属性含义：
- header ：指向跳跃表的表头节点。
- tail ：指向跳跃表的表尾节点。
- level ：记录目前跳跃表内，层数最大的那个节点的层数（表头节点的层数不计算在内）。
- length ：记录跳跃表的长度，也即是，跳跃表目前包含节点的数量（表头节点不计算在内）。

下面在重新祭出那张跳跃表的定义图：
{% asset_img skiplist.png %}

## 跳跃表操作
### 创建跳跃表
```C
zskiplist *zslCreate(void) {
    int j;
    zskiplist *zsl;
    //申请跳表内存空间
    zsl = zmalloc(sizeof(*zsl));
    //由于当前还没有任何节点，初始化为1层。
    zsl->level = 1;
    //当前跳表中的节点长度为0
    zsl->length = 0;
    //创建一个层数为32的头结点
    zsl->header = zslCreateNode(ZSKIPLIST_MAXLEVEL,0,NULL);
    //将头结点的每一层forward指针置为NULL
    for (j = 0; j < ZSKIPLIST_MAXLEVEL; j++) {
        zsl->header->level[j].forward = NULL;
        zsl->header->level[j].span = 0;
    }
    //将头结点的前置节点指针置为NULL
    zsl->header->backward = NULL;
    //跳表的tail指针置为NULL
    zsl->tail = NULL;
    return zsl;
}
```
创建一个跳表的过程很简单，仅仅完成了一件事，就是初始化头结点。

### 插入节点
首先定义一个方便描述的名词：
层标记节点：由上一层降到下一层的那个交叉节点。
排名：首节点到层标记节点的跨度（首节点与层标记节点之间有多少个节点）
```C
zskiplistNode *zslInsert(zskiplist *zsl, double score, sds ele) {
    /**
     * update:记录每一层的层标记节点
     */
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    /**
     * rank：记录排名
     */
    unsigned int rank[ZSKIPLIST_MAXLEVEL];
    int i, level;

    serverAssert(!isnan(score));
    x = zsl->header;
    /**
     * 先从最高层开始遍历
     * 循环结束后的状态一定是：
     * 终止于第一层
     */
    for (i = zsl->level-1; i >= 0; i--) {
        /* store rank that is crossed to reach the insert position */
        /**
         * 最高层的跨度为初始化为0
         */
        rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];
        /**
         * 对于一层的查找终止条件是：
         * 需要插入的元素score 小于 当前遍历的元素score。
         */
        while (x->level[i].forward &&
                (x->level[i].forward->score < score ||
                    (x->level[i].forward->score == score &&
                    sdscmp(x->level[i].forward->ele,ele) < 0)))
        {
            // 跨度叠加
            rank[i] += x->level[i].span;
            //元素移到下一位
            x = x->level[i].forward;
        }
        //记录层标记节点。
        update[i] = x;
    }
    /* we assume the element is not already inside, since we allow duplicated
     * scores, reinserting the same element should never happen since the
     * caller of zslInsert() should test in the hash table if the element is
     * already inside or not. */
    //随机分配该节点的层数
    level = zslRandomLevel();
    //如果该节点的层数大于目前的最大节点，就需要变更：
    //1、当前的最大节点
    //2、使得第一个节点的当前最大层的下一个节点指向该节点
    if (level > zsl->level) {
        for (i = zsl->level; i < level; i++) {
            rank[i] = 0;
            update[i] = zsl->header;
            update[i]->level[i].span = zsl->length;
        }
        zsl->level = level;
    }
    //创建该节点
    x = zslCreateNode(level,score,ele);
    //将上述计算出来的各层的标记点的forward指针指向该节点的各层
    for (i = 0; i < level; i++) {
        x->level[i].forward = update[i]->level[i].forward;
        update[i]->level[i].forward = x;

        /* update span covered by update[i] as x is inserted here */
        x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);
        update[i]->level[i].span = (rank[0] - rank[i]) + 1;
    }

    /* increment span for untouched levels */
    for (i = level; i < zsl->level; i++) {
        update[i]->level[i].span++;
    }

    x->backward = (update[0] == zsl->header) ? NULL : update[0];
    if (x->level[0].forward)
        x->level[0].forward->backward = x;
    else
        zsl->tail = x;
    zsl->length++;
    return x;
}
```
下面简单的描述一下，插入逻辑（更加细致的过程可以阅读这篇博文：http://www.leoox.com/?p=347）：
1、由于当前跳表记录了当前层数最高的那个节点的高度，所以可以直接从层数最高的节点开始遍历。
2、如果当前节点小于要插入的值，那就说明，要插入的值在该值的右面，再往后遍历就存在两种可能：同层右面有节点和同层右侧无节点，如果是前者则进行第3步，否则进行第3步。
3、那就直接与同层右侧的节点值进行比较。
4、如果右侧没有节点，就会进行降层，从上一层将到下一层，并将该节点记录到update[x]=item中，其中x为层数。
5、最终总会遍历到第一层，在这过程中，会将每一层降层的那个节点记录到update数组中。
6、创建一个新的节点，新创建的节点层数是随机的。
7、将update[0...i]每层分别指向新创建的节点。
8、完成节点的插入。
