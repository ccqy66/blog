---
title: Redis源码阅读-内存淘汰机制
date: 2018-05-29 19:58:26
tags:
- 源码
- Redis
- 内存淘汰机制
categories:
- Redis
---

## 内存淘汰机制

​     Redis作为一个内存数据库，并且内存在计算机中属于一种昂贵且有限的资源。随着Redis的运行，总会达到一种状态：Redis数据库装满了所有的内存。此时，虽然操作系统可以利用虚拟内存解决这种情况。但是使用虚拟内存会发生swap，频繁的swap会导致性能的开销。所以Redis实现了当内存不足时，淘汰旧key的策略。

<!--more-->

## 配置

​    Redis提供了8种关于内存淘汰相关的配置，且配置项为`maxmemory-policy`，下面分别列出这8种配置分别提供的功能：

| 配置项             | 含义                       |
| --------------- | ------------------------ |
| volatile-lru    | 从过期键数据库中使用近似LRU算法淘汰键     |
| allkeys-lru     | 从所有键数据中使用近似LRU算法淘汰键      |
| volatile-lfu    | 从过期键数据库中使用近似LFU算法淘汰键     |
| allkeys-lfu     | 从所有键数据中使用近似LFU算法淘汰键      |
| volatile-random | 从过期键数据库中随机淘汰键            |
| allkeys-random  | 从所有键数据中随机淘汰键             |
| volatile-ttl    | 移除最近过期的键                 |
| noeviction      | 不使用淘汰机制。当内存不足时可能会抛出OOM异常 |

## 实现原理

​     关于原理的解释，本章节打算从如下方面入手：何时触发淘汰、怎么触发淘汰。

#### 何时触发

​    Redis每次执行命令时，都会判断当前服务器的内存使用量是否大于配置的最大内存`maxmemory`，只有当前的内存使用量大于`maxmemory`才会执行淘汰。

```c
if (server.maxmemory) {
    int retval = freeMemoryIfNeeded();
    /* freeMemoryIfNeeded may flush slave output buffers. This may result
     * into a slave, that may be the active client, to be freed. */
    if (server.current_client == NULL) return C_ERR;
    /* It was impossible to free enough memory, and the command the client
     * is trying to execute is denied during OOM conditions? Error. */
    if ((c->cmd->flags & CMD_DENYOOM) && retval == C_ERR) {
        flagTransaction(c);
        addReply(c, shared.oomerr);
        return C_OK;
    }
}
```

#### 如何触发

​    所有逻辑都在`freeMemoryIfNeeded`   函数中完成的，下面通过一个流程图来了解该逻辑中主要完成哪些工作？

```flow
en=>inputoutput: 结束
oom=>inputoutput: 释放内存不足，抛出OOM异常
free_memory_for_startegy=>operation: 按照策略释放内存
to_pool=>operation: 将数据库中的键放入到采样池中
find_evict=>operation: 从采样池中找到一个最优淘汰键
find_random=>operation: 从数据库中随机选择一个键
is_free_condition=>condition: 释放内存释放>=mem_tofree？
is_no_random=>condition: 策略为随机模式？
tofree=>operation: 计算需要释放内存大小：mem_tofree
is_no_evict=>condition: 策略为noeviction?
free_bio=>operation: 释放lazyfree任务队列内存
free_memory=>operation: 释放该选择出键的内存空间
tofree->is_no_evict
is_no_evict(yes,right)->free_bio->oom
is_no_evict(no,left)->free_memory_for_startegy->is_free_condition
is_free_condition(yes,right)->en
is_free_condition(no)->is_no_random
is_no_random(no)->to_pool->find_evict->free_memory
is_no_random(yes)->find_random->free_memory
free_memory(left)->is_free_condition
```

根据上述的流程图，我们对着源码慢慢分析：

-   策略为noeviction时，换句话说就是不使用任何淘汰机制时：

```c
if (server.maxmemory_policy == MAXMEMORY_NO_EVICTION)
    goto cant_free; /* We need to free memory, but policy forbids. */
......
cant_free:
    /* We are here if we are not able to reclaim memory. There is only one
     * last thing we can try: check if the lazyfree thread has jobs in queue
     * and wait... */
    while(bioPendingJobsOfType(BIO_LAZY_FREE)) {
        if (((mem_reported - zmalloc_used_memory()) + mem_freed) >= mem_tofree)
            break;
        usleep(1000);
    }
    return C_ERR;
```

当策略为`noeviction`时，唯一能做的就是等待后台线程执行layfree释放内存空间。如果完成了所有的后台layfree任务，内存还不足的话，那只有抛出`OOM`异常了。

-   策略为非noeviction时，将会按照不同的策略淘汰不同键值。直至释放的内存足够存放当前键为止。需要注意一点的是：每次循环淘汰过程中，都会构建一个候选池。

    -   候选池的构建：

       {% asset_img pool.png %}

      对象信息：

    ```C
    lru字段
    |----------|----------|
    |->24bits<-|->8bits <-|
    |对象访问时间|LFU访问频率|
    typedef struct redisObject {
       。。。
       unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                                * LFU data (least significant 8 bits frequency
                                * and most significant 16 bits decreas time). */
    }
    ```

    这个所谓的候选池实际上要做的就是根据不同策略的特征，构建一个对象空闲时间升序的序列。例如：LRU策略构建出的候选池实际上就是根据LRU时钟计算的空间时间升序的对象序列。所选择出来的对象都是按照idle升序排列的。

    -   LFU策略：

        LFU的淘汰策略是LRU策略的一种优化，由于LRU策略仅仅依赖上一次访问的时间，这就会导致一个问题：

        -   本次距离上一次访问时间很短（idle时间短），不会被淘汰，但是不能保证该键是后续的热点键。
        -   本次距离上一次访问时间很长（idle时间长），会被淘汰，但是不能保证该键后续不会成为热点键。

        鉴于以上问题，redis作者实现了一种渐进LRU算法，称为LFU（最近最频繁使用算法），但是改实现并非是重新创建一个数据结构，而是利用算法，并重用当前对象的lru字段。使用8位来表示过去一段时间内的访问频率。由于8位最大只能表达255，但是通过随机函数，可能保证该值并不会很快达到上限：

        ```C
        uint8_t LFULogIncr(uint8_t counter) {
            if (counter == 255) return 255;
            double r = (double)rand()/RAND_MAX;
            double baseval = counter - LFU_INIT_VAL;
            if (baseval < 0) baseval = 0;
            double p = 1.0/(baseval*server.lfu_log_factor+1);
            if (r < p) counter++;
            return counter;
        }
        ```

        抽象出函数表达式：$p=\frac{1}{(counter-5)*factor+1}$

        代入系统默认值：$p=\frac{1}{(counter-5)*10-1}$

        绘制出函数图，如下所示：

        {% asset_img save.svg %}

        得出经验值：

        ```
        +--------+------------+------------+------------+------------+------------+
        # | factor | 100 hits   | 1000 hits  | 100K hits  | 1M hits    | 10M hits   |
        # +--------+------------+------------+------------+------------+------------+
        # | 0      | 104        | 255        | 255        | 255        | 255        |
        # +--------+------------+------------+------------+------------+------------+
        # | 1      | 18         | 49         | 255        | 255        | 255        |
        # +--------+------------+------------+------------+------------+------------+
        # | 10     | 10         | 18         | 142        | 255        | 255        |
        # +--------+------------+------------+------------+------------+------------+
        # | 100    | 8          | 11         | 49         | 143        | 255        |
        # +--------+------------+------------+------------+------------+------------+
        ```

        当counter值达到255时，数值将不会上升，当服务运行一段时间后，大量的键都会达到这个数，这样就会导致区分度降低，无法正确满足淘汰策略条件。为此需要增加值减少的策略，当键一段时间时间不访问时，会根据策略减低该值的大小：

        ```c
        unsigned long LFUDecrAndReturn(robj *o) {
            unsigned long ldt = o->lru >> 8;
            unsigned long counter = o->lru & 255;
            /**
             * 如果一个对象
             */
            if (LFUTimeElapsed(ldt) >= server.lfu_decay_time && counter) {
                if (counter > LFU_INIT_VAL*2) {
                    counter /= 2;
                    if (counter < LFU_INIT_VAL*2) counter = LFU_INIT_VAL*2;
                } else {
                    counter--;
                }
                o->lru = (LFUGetTimeInMinutes()<<8) | counter;
            }
            return counter;
        }
        ```


    -   键值淘汰

        当构建出了淘汰候选池，剩下的就是根据候选池的顺序，淘汰键值：

        ```C
        if (bestkey) {
                    db = server.db+bestdbid;
                    robj *keyobj = createStringObject(bestkey,sdslen(bestkey));
                    propagateExpire(db,keyobj,server.lazyfree_lazy_eviction);
                    delta = (long long) zmalloc_used_memory();
                    latencyStartMonitor(eviction_latency);
                    if (server.lazyfree_lazy_eviction)
                        dbAsyncDelete(db,keyobj);
                    else
                        dbSyncDelete(db,keyobj);
                    latencyEndMonitor(eviction_latency);
                    latencyAddSampleIfNeeded("eviction-del",eviction_latency);
                    latencyRemoveNestedEvent(latency,eviction_latency);
                    delta -= (long long) zmalloc_used_memory();
                    mem_freed += delta;
                    server.stat_evictedkeys++;
                    notifyKeyspaceEvent(NOTIFY_EVICTED, "evicted",
                        keyobj, db->id);
                    decrRefCount(keyobj);
                    keys_freed++;
                    if (slaves) flushSlavesOutputBuffers();
                    if (server.lazyfree_lazy_eviction && !(keys_freed % 16)) {
                        overhead = freeMemoryGetNotCountedMemory();
                        mem_used = zmalloc_used_memory();
                        mem_used = (mem_used > overhead) ? mem_used-overhead : 0;
                        if (mem_used <= server.maxmemory) {
                            mem_freed = mem_tofree;
                        }
                    }
                }
        ```

        bestkey就是从候选池中选择出来最优淘汰的键。

## 总结

-   淘汰一个键值，首先会从全量键中，随机选择`maxmemory-samples`个键，并构造一个候选池，候选池中的键是按照键的idle时间进行排序的，但是不同的策略，idle时间的计算算法不同。

    -   lru策略的算法为：键每次访问都会更新对象的lru字段，故计算方式为：

        (LRU_CLOCK() - o->lru)*LRU_CLOCK_RESOLUTION

    -   lfu策略的算法为：根据一个渐进算法，计算访问次数，该次数包含在lru字段的后8位。该值越大，表示访问越频繁，对应的idle时间也就越小。

    -   ttl策略的算法为：根据键的过期时间，过期时间越短，认为idle时间越大。



<script type="text/javascript"
  src="http://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML"></script>
