---
title: Redis源码阅读-事务
date: 2018-04-24 19:44:05
tags:
- 源码
- Redis
- 事务
categories:
- Redis
---
## 事务
&emsp;&emsp;Redis通过`MULTI`/`EXEC`/`DISCARD`和`WATCH`四个命令来实现事务功能。但是Redis的事务实现与一般的数据库事务实现不同，Redis的事务仅仅保证：整个事务期间命令执行的顺序一定与提交顺序相同。并不像其他数据库一样保证ACID。
<!--more-->
## 为啥需要
考虑一下，我们有这样的一个场景：
存在如下三个命令：
1、`SET s1 turn_on`
2、`INCR number`
3、`GET status`
要求这三个命令执行的顺序为`1、2、3`,且在执行过程中，不能有其他命令的执行。因为Redis是单线程的，一次只能执行一条命令，所以当有多个客户端连接时，虽然可以保证顺序的执行可以按照`1、2、3`来执行，但是并不能保证在三条命令执行的过程中，服务是否会执行其他客户端提交的命令。此时就需要有一个机制保证：

- 按顺序执行多个命令。
- 服务器在执行多个命令的过程中不会被中断并执行其他命令。

## 如何使用
如果我们要保证命令执行的顺序性，例如要将下列三个命令要按照顺序执行：
- `SET PRICE 100`
- `INCR NAME`
- `GET NAME`

执行时一定要保证这个顺序，且在执行过程中不允许有其他命令的加塞。此时我们就可以通过Redis的事务来实现。具体实现如下：
1、开启一个事务：
```C
127.0.0.1:6379> multi
OK
```
2、第一个命令：
```C
127.0.0.1:6379> set price 100
QUEUED
```
3、第二个命令：
```C
127.0.0.1:6379> incr price
QUEUED
```
4、第三个命令：
```C
127.0.0.1:6379> get price
QUEUED
```
5、执行命令：
```C
127.0.0.1:6379> exec
1) OK
2) (integer) 101
3) "101"
```

## 原理实现
Redis事务的使用从开始到结束通常会经历三个步骤：
- 事务开始
- 命令入队列
- 事务执行

### 事务开始

```C
void multiCommand(client *c) {
    if (c->flags & CLIENT_MULTI) {
        addReplyError(c,"MULTI calls can not be nested");
        return;
    }
    c->flags |= CLIENT_MULTI;
    addReply(c,shared.ok);
}
```
事务开始部分很简单，就是标记当前客户端为事务状态：flags|=CLIENT_MULTI，如果当前客户端已经处于事务状态，那么该命令会返回一个错误信息，且并不会退出事务状态。

### 命令入队列

```C
int processCommand(client *c) {
   .....
   /* Exec the command */
   if (c->flags & CLIENT_MULTI &&
       c->cmd->proc != execCommand && c->cmd->proc != discardCommand &&
       c->cmd->proc != multiCommand && c->cmd->proc != watchCommand)
   {
       queueMultiCommand(c);
       addReply(c,shared.queued);
   } else {
       call(c,CMD_CALL_FULL);
       c->woff = server.master_repl_offset;
       if (listLength(server.ready_keys))
           handleClientsBlockedOnLists();
   }
  .....
}
```
该段逻辑首先会判断当前客户端所处的状态flags（是否为事务状态），如果当前客户端正处于事务状态且当前执行的命令不为下述四个命令：`EXEC/DISCARD/MULTI/WATCH`则将当前命令入队列，否则的话就正常执行命令。
在分析入队列逻辑之前，我们需要了解一下队列中元素的数据结构：
```C
typedef struct multiCmd {
    robj **argv;              /* 参数列表 */
    int argc;                 /* 参数的数量 */
    struct redisCommand *cmd; /* 命令 */
} multiCmd;

typedef struct multiState {
    multiCmd *commands;     /* 保存事务命令的数组 */
    int count;              /* 命令的数量 */
    int minreplicas;        /* MINREPLICAS for synchronous replication */
    time_t minreplicas_timeout; /* MINREPLICAS timeout as unixtime. */
} multiState;
```

入队列逻辑：
```C
/* Add a new command into the MULTI commands queue */
void queueMultiCommand(client *c) {
    multiCmd *mc;
    int j;

    c->mstate.commands = zrealloc(c->mstate.commands,
            sizeof(multiCmd)*(c->mstate.count+1));
    mc = c->mstate.commands+c->mstate.count;
    mc->cmd = c->cmd;
    mc->argc = c->argc;
    mc->argv = zmalloc(sizeof(robj*)*c->argc);
    memcpy(mc->argv,c->argv,sizeof(robj*)*c->argc);
    for (j = 0; j < c->argc; j++)
        incrRefCount(mc->argv[j]);
    c->mstate.count++;
}
```

每个客户端结构中都有一个`mstate`的属性，该字段保存了所有事务过程中的命令，实际上就是`multiState`结构体。添加一条新的命令，实际上就是在commands数组中添加一个新的元素。

### WATCH命令
在命令执行之前，可以通过`watch`监听多个数据库的键。当执行`EXEC`命令时，会检查所监听的键是否有至少一个变更。如果从事务开始之后到事务执行之时，监听的键至少有一个发生变更，就会终止事务的执行。该命令实际上就是一个`乐观锁`。
```C
void watchCommand(client *c) {
    int j;

    if (c->flags & CLIENT_MULTI) {
        addReplyError(c,"WATCH inside MULTI is not allowed");
        return;
    }
    for (j = 1; j < c->argc; j++)
        watchForKey(c,c->argv[j]);
    addReply(c,shared.ok);
}
/* Watch for the specified key */
void watchForKey(client *c, robj *key) {
    list *clients = NULL;
    listIter li;
    listNode *ln;
    watchedKey *wk;

    /* Check if we are already watching for this key */
    /**
     * 首先检该键是否已经被监听了，如果已经被监听，就不需要处理了，直接返回
     */
    listRewind(c->watched_keys,&li);
    while((ln = listNext(&li))) {
        wk = listNodeValue(ln);
        if (wk->db == c->db && equalStringObjects(key,wk->key))
            return; /* Key already watched */
    }
    /* This key is not already watched in this DB. Let's add it */
    /**
     * 然后检查，该键是否存在于数据库中的watched_keys字典字段中，如果不存在，就添加到字典中
     * 该字典中的存储结构为：
     * key=>{list:[clients]}
     */
    clients = dictFetchValue(c->db->watched_keys,key);
    if (!clients) {
        clients = listCreate();
        dictAdd(c->db->watched_keys,key,clients);
        incrRefCount(key);
    }
    /**
     * 将当前客户端添加到该key对应的客户端列表
     */
    listAddNodeTail(clients,c);
    /* Add the new key to the list of keys watched by this client */
    wk = zmalloc(sizeof(*wk));
    wk->key = key;
    wk->db = c->db;
    incrRefCount(key);
    /**
     * 将当前key添加当前客户端的watched_keys属性中
     */
    listAddNodeTail(c->watched_keys,wk);
}
```
为了实现`WATCH`功能，需要两个结构：
```C
typedef struct redisDb {
    ...
    dict *watched_keys;         /* WATCHED keys for MULTI/EXEC CAS */
    ...
} redisDb;
typedef struct client {
    ...
    list *watched_keys;     /* Keys WATCHED for MULTI/EXEC CAS */
    ...
} client;
```
- `redisDb`结构中的`watched_keys`字典记录了该数据库被监听的键与客户端的关系，并通过该结构服务器可以知道有哪些键被监听了，并且监听该键的有哪些客户端。
- `client`结构中的`watched_keys`列表记录了该客户端所需要监听的键。

### 命令执行

```c
void execCommand(client *c) {
    int j;
    robj **orig_argv;
    int orig_argc;
    struct redisCommand *orig_cmd;
    int must_propagate = 0; /* Need to propagate MULTI/EXEC to AOF / slaves? */
    int was_master = server.masterhost == NULL;
    /**
     * 如果当前客户端不是处于事务状态，执行该命令将会直接报错
     */
    if (!(c->flags & CLIENT_MULTI)) {
        addReplyError(c,"EXEC without MULTI");
        return;
    }
    /**
     * 检查一下是否需要取消执行EXEC，如果存在下述情况：
     * 1、被监听的键被修改了。
     * 2、入队列时出现问题。
     */
    /* Check if we need to abort the EXEC because:
     * 1) Some WATCHed key was touched.
     * 2) There was a previous error while queueing commands.
     * A failed EXEC in the first case returns a multi bulk nil object
     * (technically it is not an error but a special behavior), while
     * in the second an EXECABORT error is returned. */
    if (c->flags & (CLIENT_DIRTY_CAS|CLIENT_DIRTY_EXEC)) {
        addReply(c, c->flags & CLIENT_DIRTY_EXEC ? shared.execaborterr :
                                                  shared.nullmultibulk);
        discardTransaction(c);
        goto handle_monitor;
    }

    /* Exec all the queued commands */
    unwatchAllKeys(c); /* Unwatch ASAP otherwise we'll waste CPU cycles */
    orig_argv = c->argv;
    orig_argc = c->argc;
    orig_cmd = c->cmd;
    addReplyMultiBulkLen(c,c->mstate.count);
    for (j = 0; j < c->mstate.count; j++) {
        c->argc = c->mstate.commands[j].argc;
        c->argv = c->mstate.commands[j].argv;
        c->cmd = c->mstate.commands[j].cmd;

        /* Propagate a MULTI request once we encounter the first command which
         * is not readonly nor an administrative one.
         * This way we'll deliver the MULTI/..../EXEC block as a whole and
         * both the AOF and the replication link will have the same consistency
         * and atomicity guarantees. */
        if (!must_propagate && !(c->cmd->flags & (CMD_READONLY|CMD_ADMIN))) {
            execCommandPropagateMulti(c);
            must_propagate = 1;
        }

        call(c,CMD_CALL_FULL);

        /* Commands may alter argc/argv, restore mstate. */
        c->mstate.commands[j].argc = c->argc;
        c->mstate.commands[j].argv = c->argv;
        c->mstate.commands[j].cmd = c->cmd;
    }
    c->argv = orig_argv;
    c->argc = orig_argc;
    c->cmd = orig_cmd;
    discardTransaction(c);

    /* Make sure the EXEC command will be propagated as well if MULTI
     * was already propagated. */
    if (must_propagate) {
        int is_master = server.masterhost == NULL;
        server.dirty++;
        /* If inside the MULTI/EXEC block this instance was suddenly
         * switched from master to slave (using the SLAVEOF command), the
         * initial MULTI was propagated into the replication backlog, but the
         * rest was not. We need to make sure to at least terminate the
         * backlog with the final EXEC. */
        if (server.repl_backlog && was_master && !is_master) {
            char *execcmd = "*1\r\n$4\r\nEXEC\r\n";
            feedReplicationBacklog(execcmd,strlen(execcmd));
        }
    }

handle_monitor:
    /* Send EXEC to clients waiting data from MONITOR. We do it here
     * since the natural order of commands execution is actually:
     * MUTLI, EXEC, ... commands inside transaction ...
     * Instead EXEC is flagged as CMD_SKIP_MONITOR in the command
     * table, and we do it here with correct ordering. */
    if (listLength(server.monitors) && !server.loading)
        replicationFeedMonitors(c,server.monitors,c->db->id,c->argv,c->argc);
}
```
在之前的基础之上再分析事务的执行过程就比较简单了，执行过程实际上就是执行进入到队列里的命令，同时，如果之前使用了`WATCH`命令监听了若干个键，并且这些键中的一个或多个在执行之前发生了变更，就会终止事务的执行。

## 总结
- Redis的事务是具有原子性的，因为进入到事务队列中的命令，要么全部执行，要么全部不执行。事务不执行存在两种情况：
  - 命令入事务队列过程中出现错误。
  - 被`WATCH`的键在执行事务之前被修改了。
- Redis的事务不支持回滚操作，也就是说，只要事务进入了执行过程，即便是其中有一个命令执行失败，事务也不会终止，会一直运行下去。
- 由于Redis是单线程的，所有Redis天然的满足事务的隔离性。
- 事务提供了一种将多个命令打包，然后一次性、有序地执行的机制。
- 多个命令会被入到事务队列中，然后按先进先出的顺序执行。
- 事务执行过程中是不会被中断的（因为Redis的单线程的），当事务队列中的所有名录库都执行完毕后，事务才会结束。
- Redis事务支持ACI，只有当服务器开启aof时，Redis的事务才支持D，持久性。
