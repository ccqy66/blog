---
title: Redis源码阅读-字符串命令
date: 2018-03-29 17:46:48
tags:
- 源码
- Redis
- 字符串命令
categories:
- Redis
---
## 字符串命令
&emsp;&emsp;本文将详细分析字符串相关的命令
<!--more-->
## 命令
### GET
命令入口为：t_string.c/getCommand
主要逻辑为：db.c/lookupKey
```C
robj *lookupKey(redisDb *db, robj *key, int flags) {
    //从数据库字典中查找指定key的value值
    dictEntry *de = dictFind(db->dict,key->ptr);
    if (de) {
        //获取节点的值
        robj *val = dictGetVal(de);
        //更新lru值
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
该命令的执行逻辑很简单：由于Redis将所有的key-value都是保存在底层为字典的数据结构中，get命令实际上就是从这个字典中查找指定key的value。

### SET
命令入口：t_string.c/setCommand
主要逻辑：t_string.c/setGenericCommand 和 db.c/setKey
```C
void setGenericCommand(client *c, int flags, robj *key, robj *val, robj *expire, int unit, robj *ok_reply, robj *abort_reply) {
    long long milliseconds = 0; /* initialized to avoid any harmness warning */
    //如果设置了过期时间，校验过期时间并进行单位统一
    if (expire) {
        //如果从对象中获取过期时间失败，则直接返回失败
        if (getLongLongFromObjectOrReply(c, expire, &milliseconds, NULL) != C_OK)
            return;
        //如果过期时间 <= 0 ，为不合法的过期时间
        if (milliseconds <= 0) {
            addReplyErrorFormat(c,"invalid expire time in %s",c->cmd->name);
            return;
        }
        //单位换算
        if (unit == UNIT_SECONDS) milliseconds *= 1000;
    }
    // 如果设置操作标记为NX，并且该key存在
    // 或者这只操作标记为XX，并且该key不存在
    // 直接返回
    if ((flags & OBJ_SET_NX && lookupKeyWrite(c->db,key) != NULL) ||
        (flags & OBJ_SET_XX && lookupKeyWrite(c->db,key) == NULL))
    {
        addReply(c, abort_reply ? abort_reply : shared.nullbulk);
        return;
    }
    //将该key写入到数据库字典中
    setKey(c->db,key,val);
    //dirty字段+1
    server.dirty++;
    //如果设置了过期时间，就将过期时间添加到过期时间字典中
    if (expire) setExpire(c,c->db,key,mstime()+milliseconds);
    notifyKeyspaceEvent(NOTIFY_STRING,"set",key,c->db->id);
    if (expire) notifyKeyspaceEvent(NOTIFY_GENERIC,
        "expire",key,c->db->id);
    addReply(c, ok_reply ? ok_reply : shared.ok);
}
void setKey(redisDb *db, robj *key, robj *val) {
    //如果数据库字典中不存在，调用add
    if (lookupKeyWrite(db,key) == NULL) {
        dbAdd(db,key,val);
    } else {
        //如果已经存在，就调用覆盖
        dbOverwrite(db,key,val);
    }
    //value对象的引用计数加1
    incrRefCount(val);
    //清除之前键值的过期时间
    removeExpire(db,key);
    signalModifiedKey(db,key);
}
```
该命令同时实现了SETNX/SETXX/SETEX/SETPX4个命令的功能，并且逻辑也很简单，并没有什么可讲的，都在注释中。

### GETBIT
命令: GETBIT key offset
代码：bitops.c/getbitCommand

```
void getbitCommand(client *c) {
    robj *o;
    char llbuf[32];
    size_t bitoffset;
    size_t byte, bit;
    size_t bitval = 0;
    //获取参数 offset的值
    if (getBitOffsetFromArgument(c,c->argv[2],&bitoffset,0,0) != C_OK)
        return;
    //查询key键的value值
    if ((o = lookupKeyReadOrReply(c,c->argv[1],shared.czero)) == NULL ||
        checkType(c,o,OBJ_STRING)) return;
    //获取目标offset在第几个字节位上
    byte = bitoffset >> 3;
    //bitoffset & 0x7 相当于 bitoffset%8
    //7-(bitoffset & 0x7)：计算在一个字节上从右向左的位置，便于左移求掩码
    bit = 7 - (bitoffset & 0x7);
    //字符串有3中编码形式，不同编码的处理形式不一样
    //判断是否为字符串编码
    if (sdsEncodedObject(o)) {
        if (byte < sdslen(o->ptr))
            //通过掩码bit取得指定位上的值
            bitval = ((uint8_t*)o->ptr)[byte] & (1 << bit);
    } else {
        //如果是数字编码的话
        if (byte < (size_t)ll2string(llbuf,sizeof(llbuf),(long)o->ptr))
            bitval = llbuf[byte] & (1 << bit);
    }
    addReply(c, bitval ? shared.cone : shared.czero);
}
```
取得指定偏移量上的位上有以下流程：
1、首先偏移量除以8，求得该偏移量在第几个字节上：bitoffset >> 3
2、然后求得该偏移量在指定字节上的第几位（从右向左算）：7 - (bitoffset & 0x7)
3、通过2过程计算的位置，获取该偏移位对应数值：((uint8_t*)o->ptr)[byte] & (1 << bit);
4、如果对应的数值等于0表示该偏移量上为0，否则为1。

### INCR
命令：INCR key
代码：t_string.c/incrDecrCommand
```C
void incrDecrCommand(client *c, long long incr) {
    long long value, oldvalue;
    robj *o, *new;

    //从数据库中查找对应key的数据
      o = lookupKeyWrite(c->db,c->argv[1]);
      //如果存在，但不是STRING类型，就直接返回
      if (o != NULL && checkType(c,o,OBJ_STRING)) return;
      //将字符串转成longlong类型
      if (getLongLongFromObjectOrReply(c,o,&value,NULL) != C_OK) return;

      oldvalue = value;
      //判断操作后是否可能导致移除
      if ((incr < 0 && oldvalue < 0 && incr < (LLONG_MIN-oldvalue)) ||
          (incr > 0 && oldvalue > 0 && incr > (LLONG_MAX-oldvalue))) {
          addReplyError(c,"increment or decrement would overflow");
          return;
      }
      value += incr;
      //将修改后的值重置到数据库中
      //如果不存在就添加到数据库中
    if (o && o->refcount == 1 && o->encoding == OBJ_ENCODING_INT &&
        (value < 0 || value >= OBJ_SHARED_INTEGERS) &&
        value >= LONG_MIN && value <= LONG_MAX)
    {
        new = o;
        o->ptr = (void*)((long)value);
    } else {
        new = createStringObjectFromLongLong(value);
        if (o) {
            dbOverwrite(c->db,c->argv[1],new);
        } else {
            dbAdd(c->db,c->argv[1],new);
        }
    }
    signalModifiedKey(c->db,c->argv[1]);
    notifyKeyspaceEvent(NOTIFY_STRING,"incrby",c->argv[1],c->db->id);
    server.dirty++;
    addReply(c,shared.colon);
    addReply(c,new);
    addReply(c,shared.crlf);
}
```
