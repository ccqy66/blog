---
title: Redis源码阅读-数据库
date: 2018-04-04 16:09:31
tags:
- 源码
- Redis
- 对象系统
categories:
- Redis
---

## 数据库
本文将详细介绍Redis的内存数据库的表示，以及数据库的常见操作，主要包含以下几个方面
- 数据库数据结构定义
- 数据库的常见操作
- 数据库的键过期机制

<!--more-->
### 数据结构
```C
typedef struct redisDb {
    dict *dict;                 /* The keyspace for this DB */
    dict *expires;              /* Timeout of keys with a timeout set */
    dict *blocking_keys;        /* Keys with clients waiting for data (BLPOP)*/
    dict *ready_keys;           /* Blocked keys that received a PUSH */
    dict *watched_keys;         /* WATCHED keys for MULTI/EXEC CAS */
    int id;                     /* Database ID */
    long long avg_ttl;          /* Average TTL, just for stats */
} redisDb;
```
各字段含义：
- dict：该数据库的键空间，保存key-value。
- expires：保存键的过期时间，键的过期时间相关的数据保存在单独的字典中。
- blocking_keys：当执行`BLPOP`命令时，客户端等待的key列表。
- ready_keys：接收到PUSH的阻塞键。
- watched_keys：执行MULTI/EXEC原子操作监听的key。
- id：数据库的id，默认为0号数据库。
- avg_ttl：

### 常见操作
#### lookupKey
该函数是偏底层的api，在该函数中没有额外的操作，仅仅是从数据库字典中查询键并返回。被多个`lookupKey`开头函数使用。
```C
robj *lookupKey(redisDb *db, robj *key, int flags) {
    //从数据库字典中查找指定key的value值
    dictEntry *de = dictFind(db->dict,key->ptr);
    if (de) {
        //获取节点的值
        robj *val = dictGetVal(de);
        //更新lru值
        /*
         * 如果存在如下条件之一就不会更新该value对象的lru属性：
         * 1、当前正在执行rdb文件写入
         * 2、当前正在执行aof操作
         * 3、当前操作的flag=LOOKUP_NOTOUCH
         */
        /* Update the access time for the ageing algorithm.
         * Don't do it if we have a saving child, as this will trigger
         * a copy on write madness. */
        if (server.rdb_child_pid == -1 &&
            server.aof_child_pid == -1 &&
            !(flags & LOOKUP_NOTOUCH))
        {
            if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) {
                unsigned long ldt = val->lru >> 8;
                unsigned long counter = LFULogIncr(val->lru & 255);
                val->lru = (ldt << 8) | counter;
            } else {
                val->lru = LRU_CLOCK();
            }
        }
        return val;
    } else {
        return NULL;
    }
}

```
该函数的逻辑很简单，主要是调用了字典的api。主要包含如下流程：
- 从数据库字典中查找指定key的键值。
- 如果数据库中没有改键对应的值，则直接返回NULL。否则按照策略更新查询出来的value对象的lru属性。

#### lookupKeyRead
该方法的通过调用`lookupKeyReadWithFlags`来实现读取操作的查询，在Redis中查询操作有两类：读取和写入，二种不同的类型通过flags来区分。二者有如下不同：
- LOOKUP_NONE：不使用特别的标识，相当于无意义，相对于LOOKUP_NOTOUCH而言。
- LOOKUP_NOTOUCH：不修改最后一次访问key的时间。
```C
robj *lookupKeyReadWithFlags(redisDb *db, robj *key, int flags) {
    robj *val;
    // 查看该键是否过期，如果过期返回1，否则返回0（后续将详细了解该机制）
    if (expireIfNeeded(db,key) == 1) {
        /* Key expired. If we are in the context of a master, expireIfNeeded()
         * returns 0 only when the key does not exist at all, so it's safe
         * to return NULL ASAP. */
        if (server.masterhost == NULL) return NULL;

        /* However if we are in the context of a slave, expireIfNeeded() will
         * not really try to expire the key, it only returns information
         * about the "logical" status of the key: key expiring is up to the
         * master in order to have a consistent view of master's data set.
         *
         * However, if the command caller is not the master, and as additional
         * safety measure, the command invoked is a read-only command, we can
         * safely return NULL here, and provide a more consistent behavior
         * to clients accessign expired values in a read-only fashion, that
         * will say the key as non exisitng.
         *
         * Notably this covers GETs when slaves are used to scale reads. */
        if (server.current_client &&
            server.current_client != server.master &&
            server.current_client->cmd &&
            server.current_client->cmd->flags & CMD_READONLY)
        {
            return NULL;
        }
    }
    //从数据库中查找该key
    val = lookupKey(db,key,flags);
    if (val == NULL)
        server.stat_keyspace_misses++;
    else
        server.stat_keyspace_hits++;
    return val;
}
```

### 过期机制
Redis的键值支持过期时间，其中这些键的过期时间单独保存在数据库结构中的`expires`字典中，设置了过期时间，那么Redis就必须保证在过期时间到来时删除这些键，在Redis实现中有两种形式：主动删除和被动删除。下面将分别讲述这两种不同的处理形式：
- 被动删除：就是当访问到该键时，才查询该键是否已经过期，如果过期在进行删除，如果该键一直未被访问到，不会触发被动删除。被动删除的处理逻辑在`expireIfNeeded`中定义：
```C
int expireIfNeeded(redisDb *db, robj *key) {
    //获取该键的过期时间
    mstime_t when = getExpire(db,key);
    mstime_t now;
    //该键不存在过期时间
    if (when < 0) return 0; /* No expire for this key */

    /* Don't expire anything while loading. It will be done later. */
    //如果当前数据库正在处理loading状态，暂时先不过期任何键。后续再做
    if (server.loading) return 0;

    /* If we are in the context of a Lua script, we claim that time is
     * blocked to when the Lua script started. This way a key can expire
     * only the first time it is accessed and not in the middle of the
     * script execution, making propagation to slaves / AOF consistent.
     * See issue #1525 on Github for more information. */
    now = server.lua_caller ? server.lua_time_start : mstime();

    /* If we are running in the context of a slave, return ASAP:
     * the slave key expiration is controlled by the master that will
     * send us synthesized DEL operations for expired keys.
     *
     * Still we try to return the right information to the caller,
     * that is, 0 if we think the key should be still valid, 1 if
     * we think the key is expired at this time. */
     //如果允许在从节点，只会返回是否过期，并不会删除数据，
    //删除数据的操作只会在主节点同步过来的
    if (server.masterhost != NULL) return now > when;

    /* Return when this key has not expired */
    //如果还没有过期，就直接返回了
    if (now <= when) return 0;

    /* Delete the key */
    //过期键的数量+1
    server.stat_expiredkeys++;
    //传播过期事件
    propagateExpire(db,key,server.lazyfree_lazy_expire);
    //通过键空间事件
    notifyKeyspaceEvent(NOTIFY_EXPIRED,
        "expired",key,db->id);
    /**
     * 如果是懒过期，则执行异步删除，所谓异步删除的逻辑如下：
     * 1、首先从字典中删除该键值(仅仅是逻辑删除，实际上该节点仍然会占用内存空间)，并返回该键值，并不释放内存空间。
     * 2、如果该键值的大小大于某一阈值，则执行将对象添加到过期队列中，后台任务不断从队列中释放对象的内存空间。
     * 3、如果该键值的大小小于某一阈值，则同步释放内存空间。
     *
     * 否则，则执行同步删除
     */
    return server.lazyfree_lazy_expire ? dbAsyncDelete(db,key) :
                                         dbSyncDelete(db,key);
}
```
该函数的主要逻辑就是：检查对应的键是否已经过期，如果已经过期并且在主节点就删除该键，如果在从节点只会返回是否过期。包含如下逻辑：
- 1、首先从过期字典中查询该键的过期时间。如果没有设置过过期时间，就直接返回未过期。
- 2、如果当前数据库正在执行loading操作（从aof或者rdb中加载数据），则赞不做处理，后续再进行处理，就直返返回未过期。
- 3、如果当前处于子节点，就返回当前键的过期状态，并不会删除数据，因为删除数据的操作只可能会被主节点完成，从节点只会同步主节点的数据。
- 4、走到此处，说明当前键已经过期且当前环境为主节点环境，需要执行过期键的删除。
- 5、删除之前，需要将该键的过期传播到所有从节点和aof文件中。
- 6、发送键过期事件。
- 7、如果当前Redis配置成懒过期，就执行异步删除操作，由函数`dbAsyncDelete`控制。
- 8、否则执行同步操作。

在第7步中，我们发现几个关键字“懒过期”和“异步删除”，为什么需要异步删除呢？实际上删除字典中的一个键需要两个操作：
- 1、将键从字典中删除。
- 2、释放该键的内存空间。

虽然第一步并不会对Redis造成性能上的影响（仅仅改变一些指针），但是当键值所占用的内存空间比较大时，释放内存空间可能会造成性能上的影响，可能会阻塞主线程。但是第二步对整个数据库而言完全可以异步进行，因为执行了第一步之后，该键值对Redis而言已经算是删除了，只是还占用内存空间而已。
```c
#define LAZYFREE_THRESHOLD 64
int dbAsyncDelete(redisDb *db, robj *key) {
    /* Deleting an entry from the expires dict will not free the sds of
     * the key, because it is shared with the main dictionary. */
    if (dictSize(db->expires) > 0) dictDelete(db->expires,key->ptr);

    /* If the value is composed of a few allocations, to free in a lazy way
     * is actually just slower... So under a certain limit we just free
     * the object synchronously. */
    //将键从字典中删除，并返回该键，但是此时并没有释放内存空间。
    dictEntry *de = dictUnlink(db->dict,key->ptr);
    if (de) {
        robj *val = dictGetVal(de);
        //需要计算一个释放该value需要的代价，如果val很小就没有必要执行异步删除
        size_t free_effort = lazyfreeGetFreeEffort(val);

        /* If releasing the object is too much work, let's put it into the
         * lazy free list. */
        if (free_effort > LAZYFREE_THRESHOLD) {
            atomicIncr(lazyfree_objects,1);
            //将该键值添加到异步队列中，由后台线程去释放该键值的内存空间
            bioCreateBackgroundJob(BIO_LAZY_FREE,val,NULL,NULL);
            dictSetVal(db->dict,de,NULL);
        }
    }

    /* Release the key-val pair, or just the key if we set the val
     * field to NULL in order to lazy free it later. */
    //说明该键占用的内存空间不大，没有必要使用异步删除
    //执行执行同步删除
    if (de) {
        dictFreeUnlinkedEntry(db->dict,de);
        if (server.cluster_enabled) slotToKeyDel(key);
        return 1;
    } else {
        return 0;
    }
}
```
并不是执行了`dbAsyncDelete`函数一定会进行异步删除，在该函数内部会计算对应key的value的内存大小，如果大于阈值时才考虑将该value加入到队列中，由后台线程去执行内存释放过程。如果小于阈值，就相当于执行了同步删除过程。

- 主动删除：Redis内部有定时任务，某秒执行10次，在每次执行时，会从数据库中随机的选择键，如果过期就删除该键。

```C
void activeExpireCycle(int type) {
    /* This function has some global state in order to continue the work
     * incrementally across calls. */
    /**
     * 全局变量
     * current_db：当前操作的下标
     * timelimit_exit：本次循环是否退出
     * last_fast_cycle：上一次循环开始的时间
     * iteration：
     * dbs_per_call：
     * start：
     * timelimit：
     */
    static unsigned int current_db = 0; /* Last DB tested. */
    static int timelimit_exit = 0;      /* Time limit hit in previous call? */
    static long long last_fast_cycle = 0; /* When last fast cycle ran. */

    int j, iteration = 0;
    int dbs_per_call = CRON_DBS_PER_CALL;
    long long start = ustime(), timelimit;

    /* When clients are paused the dataset should be static not just from the
     * POV of clients not being able to write, but also from the POV of
     * expires and evictions of keys not being performed. */
    if (clientsArePaused()) return;

    if (type == ACTIVE_EXPIRE_CYCLE_FAST) {
        /* Don't start a fast cycle if the previous cycle did not exited
         * for time limt. Also don't repeat a fast cycle for the same period
         * as the fast cycle total duration itself. */
        if (!timelimit_exit) return;
        if (start < last_fast_cycle + ACTIVE_EXPIRE_CYCLE_FAST_DURATION*2) return;
        last_fast_cycle = start;
    }

    /* We usually should test CRON_DBS_PER_CALL per iteration, with
     * two exceptions:
     *
     * 1) Don't test more DBs than we have.
     * 2) If last time we hit the time limit, we want to scan all DBs
     * in this iteration, as there is work to do in some DB and we don't want
     * expired keys to use memory for too much time. */
    if (dbs_per_call > server.dbnum || timelimit_exit)
        dbs_per_call = server.dbnum;

    /* We can use at max ACTIVE_EXPIRE_CYCLE_SLOW_TIME_PERC percentage of CPU time
     * per iteration. Since this function gets called with a frequency of
     * server.hz times per second, the following is the max amount of
     * microseconds we can spend in this function. */
    timelimit = 1000000*ACTIVE_EXPIRE_CYCLE_SLOW_TIME_PERC/server.hz/100;
    timelimit_exit = 0;
    if (timelimit <= 0) timelimit = 1;

    if (type == ACTIVE_EXPIRE_CYCLE_FAST)
        timelimit = ACTIVE_EXPIRE_CYCLE_FAST_DURATION; /* in microseconds. */

    for (j = 0; j < dbs_per_call; j++) {
        int expired;
        redisDb *db = server.db+(current_db % server.dbnum);

        /* Increment the DB now so we are sure if we run out of time
         * in the current DB we'll restart from the next. This allows to
         * distribute the time evenly across DBs. */
        current_db++;

        /* Continue to expire if at the end of the cycle more than 25%
         * of the keys were expired. */
        do {
            unsigned long num, slots;
            long long now, ttl_sum;
            int ttl_samples;

            /* If there is nothing to expire try next DB ASAP. */
            if ((num = dictSize(db->expires)) == 0) {
                db->avg_ttl = 0;
                break;
            }
            slots = dictSlots(db->expires);
            now = mstime();

            /* When there are less than 1% filled slots getting random
             * keys is expensive, so stop here waiting for better times...
             * The dictionary will be resized asap. */
            if (num && slots > DICT_HT_INITIAL_SIZE &&
                (num*100/slots < 1)) break;

            /* The main collection cycle. Sample random keys among keys
             * with an expire set, checking for expired ones. */
            expired = 0;
            ttl_sum = 0;
            ttl_samples = 0;

            if (num > ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP)
                num = ;

            while (num--) {
                dictEntry *de;
                long long ttl;

                if ((de = dictGetRandomKey(db->expires)) == NULL) break;
                ttl = dictGetSignedIntegerVal(de)-now;
                if (activeExpireCycleTryExpire(db,de,now)) expired++;
                if (ttl > 0) {
                    /* We want the average TTL of keys yet not expired. */
                    ttl_sum += ttl;
                    ttl_samples++;
                }
            }

            /* Update the average TTL stats for this database. */
            if (ttl_samples) {
                long long avg_ttl = ttl_sum/ttl_samples;

                /* Do a simple running average with a few samples.
                 * We just use the current estimate with a weight of 2%
                 * and the previous estimate with a weight of 98%. */
                if (db->avg_ttl == 0) db->avg_ttl = avg_ttl;
                db->avg_ttl = (db->avg_ttl/50)*49 + (avg_ttl/50);
            }

            /* We can't block forever here even if there are many keys to
             * expire. So after a given amount of milliseconds return to the
             * caller waiting for the other active expire cycle. */
            iteration++;
            if ((iteration & 0xf) == 0) { /* check once every 16 iterations. */
                long long elapsed = ustime()-start;

                latencyAddSampleIfNeeded("expire-cycle",elapsed/1000);
                if (elapsed > timelimit) timelimit_exit = 1;
            }
            if (timelimit_exit) return;
            /* We don't repeat the cycle if there are less than 25% of keys
             * found expired in the current DB. */
        } while (expired > ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP/4);
    }
}
```
该函数的代码篇幅较多，主要分析一下流程：
- 1、循环历当前Redis中的所有数据库，循环退出条件如下：
> 1、超过了本次循环的要求的最大时间
> 2、一个库的过期键少于25%
> 3、当前数据库不存在配置了过期的键
> 4、当当前数据库的过期键数量过少时。

- 2、对于每一个库而言，执行如下：
> 1、随机选取 ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP 个配置了过期的键
> 2、如果选取的键过期了，键删除该键
> 3、如果当前循环执行的时间大于一个阈值`1000000*ACTIVE_EXPIRE_CYCLE_SLOW_TIME_PERC/server.hz/100`就停止本次操作。

该方案的优点如下：
1、减少了因为过期键而带来的内存浪费（相对于被动删除而言）。
2、通过限制删除操作执行的时长和频率来减少删除操作对CPU时间的影响。
