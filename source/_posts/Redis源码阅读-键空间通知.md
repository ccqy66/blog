---
title: Redis源码阅读-键空间通知
date: 2018-04-18 19:14:52
tags:
- 源码
- Redis
- 键空间通知
categories:
- Redis
---

## 键空间通知
键空间通知使得客户端可以通过订阅频道或模式，来接收那些以某种方式改动了 Redis 数据集的事件。
例如如下事件：
- 所有修改键的命令。
- 所有接收到 LPUSH 命令的键。
- 0 号数据库中所有已过期的键。

事件通过Redis的发布订阅功能来实现的。
<!--more-->
## 原理实现
键空间通知实际上就是订阅发布功能的实践，通常情况下。客户端处理的事件通道都需要其他客户端发布或者订阅消息，而键空间的不同是：会默认为一些特定的服务端操作注册通道，并当事件发生后会主动向该通道发布信息，其他客户端只需要按照规则订阅这些消息即可。
### 事件类型
实际上，当服务端进行了一个操作，键空间通知都会发送两种不同类型的事件：
- 键空间类型：
 - 命名规范：__keyspace@<db>__:<key> <event>
 - 含义：表示<db>号数据库<key>键进行了<event>事件的操作
    - __keyspace@<db>__:<key>：表示<db>数据库<key>键。
    - <event>：进行了<event>事件操作。
 - 例子：`__keyspace@0__:mykey del`
 表示：0号数据库的`mykey`键执行了`del`操作。
- 键事件类型：
 - 命名规范：__keyevente@<db>__:<event> <key>
 - 含义：表示<db>号数据库<event>事件被执行，操作的对象键是<key>
 - 例子：__keyevent@0__:del mykey
 表示：0号数据库执行了`del`事件，并且操作的对象是`mykey`键。

 > 总结：
 > 键空间类型：该事件通道记录了数据库的键执行了哪些操作。关注于事件类型
 > 键事件类型：该事件通道记录了数据库执行的哪些操作，对应的键是什么。关注于键。

 ### 事件发布
 当数据库执行了一些操作时，会调用`notifyKeyspaceEvent`函数将两个事件发送到指定的通道中。
 ```C
 void notifyKeyspaceEvent(int type, char *event, robj *key, int dbid) {
     sds chan;
     robj *chanobj, *eventobj;
     int len = -1;
     char buf[24];

     /* If notifications for this class of events are off, return ASAP. */
     if (!(server.notify_keyspace_events & type)) return;

     eventobj = createStringObject(event,strlen(event));

     /* __keyspace@<db>__:<key> <event> notifications. */
     if (server.notify_keyspace_events & NOTIFY_KEYSPACE) {
         chan = sdsnewlen("__keyspace@",11);
         len = ll2string(buf,sizeof(buf),dbid);
         chan = sdscatlen(chan, buf, len);
         chan = sdscatlen(chan, "__:", 3);
         chan = sdscatsds(chan, key->ptr);
         chanobj = createObject(OBJ_STRING, chan);
         pubsubPublishMessage(chanobj, eventobj);
         decrRefCount(chanobj);
     }

     /* __keyevente@<db>__:<event> <key> notifications. */
     if (server.notify_keyspace_events & NOTIFY_KEYEVENT) {
         chan = sdsnewlen("__keyevent@",11);
         if (len == -1) len = ll2string(buf,sizeof(buf),dbid);
         chan = sdscatlen(chan, buf, len);
         chan = sdscatlen(chan, "__:", 3);
         chan = sdscatsds(chan, eventobj->ptr);
         chanobj = createObject(OBJ_STRING, chan);
         pubsubPublishMessage(chanobj, key);
         decrRefCount(chanobj);
     }
     decrRefCount(eventobj);
 }
 ```
 代码很简单，主要是实现了发布消息，如果需要了解订阅发布功能，可以查看之前的博客《Redis源码阅读-发布订阅》。

 ## 总结
 - 如果需要订阅某些命令执行的事件，不需要单独创建新的通道，可以直接使用键空间通知机制来监听键的变更，或者某一个命令的执行。
 - 通过键空间通知机制，有两种事件可以监听，一种是键空间类型、另一种是键事件类型。如果关注键本身的操作，可以订阅前者事件，如果关注事件本身，可以订阅后者事件。
 - 键空间通知机制的实现主要依赖于Redis的发布订阅功能。
 - 因为 Redis 目前的订阅与发布功能采取的是发送即忘（fire and forget）策略， 所以如果你的程序需要可靠事件通知（reliable notification of events）， 那么目前的键空间通知可能并不适合你： 当订阅事件的客户端断线时， 它会丢失所有在断线期间分发给它的事件。
