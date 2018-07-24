---
title: Redis源码阅读-发布订阅
date: 2018-04-13 19:01:13
tags:
- 源码
- Redis
- 发布订阅
categories:
- Redis
---
## 发布订阅
&emsp;&emsp;Redis实现了发布订阅功能，在实现中，发送者（客户端）不是将消息直接发送到特定的接收者（客户端），而是将消息发送给指定的频道，然后由频道将信息转发给所有对这个频道感兴趣的订阅者。如此实现具有下列特点：
- 发送者无须知道任何关于订阅者的信息， 而订阅者也无须知道是那个客户端给它发送信息， 它只要关注自己感兴趣的频道即可。
- 对发布者和订阅者进行解构（decoupling）， 可以极大地提高系统的扩展性（scalability）， 并得到一个更动态的网络拓扑（network topology）。

<!--more-->
上述的描述，可以简单的用下图来描述：
{% asset_img subpub01.png %}
上表可以如此描述（针对channel01通道）：
- 通道`channel01`被`client01`、`client02`所订阅。
- `client02`发布一条消息到`client01`通道中。
- 新的消息到达`client01`通道，同时触发将该消息同步到所有的订阅者`client01`、`client02`。

## 原理实现
### 通道-客户端
&emsp;&emsp;在Redis实现中，通道与客户端的关系使用字典数据结构存储的，一个通道对应一个客户端列表。如下图所示：
{% asset_img channel.png %}
在该映射中，key为通道名称，value为客户端的列表，使用列表数据结构实现。

### 订阅通道
```C
int pubsubSubscribeChannel(client *c, robj *channel) {
    dictEntry *de;
    list *clients = NULL;
    int retval = 0;

    /* Add the channel to the client -> channels hash table */
    /**
     * 当前 client->channels hash table中还不存在该频道，已经存在就不需要再处理了
     * 就在hash table中添加一个新的频道
     * 该hash table的数据结构为字典
     * key ==> channel
     * value ==> client lists
     */
    if (dictAdd(c->pubsub_channels,channel,NULL) == DICT_OK) {
        retval = 1;
        incrRefCount(channel);
        /* Add the client to the channel -> list of clients hash table */
        de = dictFind(server.pubsub_channels,channel);
        if (de == NULL) {
            clients = listCreate();
            dictAdd(server.pubsub_channels,channel,clients);
            incrRefCount(channel);
        } else {
            clients = dictGetVal(de);
        }
        listAddNodeTail(clients,c);
    }
    /* Notify the client */
    addReply(c,shared.mbulkhdr[3]);
    addReply(c,shared.subscribebulk);
    addReplyBulk(c,channel);
    addReplyLongLong(c,clientSubscriptionsCount(c));
    return retval;
}
```
Redis将频道和客户端的订阅关系保存在服务器的`pubsub_channels`字典字段中:
```C
struct redisServer{
    ...
    /* Pubsub */
    dict *pubsub_channels;  /* Map channels to list of subscribed clients */
    list *pubsub_patterns;  /* A list of pubsub_patterns */
    ...
}
```
当客户端使用`SUBSCRIBE`命令订阅一个不存在频道时，首先会在服务器的`pubsub_channels`字典中创建一个新的key-value（key为频道名，value为客户端列表），然后将当前客户端添加到该客户端列表中。
### 订阅模式通道

```C
typedef struct pubsubPattern {
    client *client;
    robj *pattern;
} pubsubPattern;

int pubsubSubscribePattern(client *c, robj *pattern) {
    int retval = 0;
    if (listSearchKey(c->pubsub_patterns,pattern) == NULL) {
        retval = 1;
        pubsubPattern *pat;
        listAddNodeTail(c->pubsub_patterns,pattern);
        incrRefCount(pattern);
        pat = zmalloc(sizeof(*pat));
        pat->pattern = getDecodedObject(pattern);
        pat->client = c;
        listAddNodeTail(server.pubsub_patterns,pat);
    }
    /* Notify the client */
    addReply(c,shared.mbulkhdr[3]);
    addReply(c,shared.psubscribebulk);
    addReplyBulk(c,pattern);
    addReplyLongLong(c,clientSubscriptionsCount(c));
    return retval;
}
```
客户端除了可以订阅一个特定的频道，还可以通过调用`PSUBSCRIBE`命令订阅一个模式。一个模式可能包含多个频道，例如存在一个模式：`com.wolfcoder.*`，那么如果一个客户端订阅了该频道就等于订阅了所有`com.wolfcoder.`开头的频道，例如`com.wolfcoder.cc`,`com.wolfcoder.cy`
### 通道发布
```C
int pubsubPublishMessage(robj *channel, robj *message) {
    int receivers = 0;
    dictEntry *de;
    listNode *ln;
    listIter li;

    /* Send to clients listening for that channel */
    de = dictFind(server.pubsub_channels,channel);
    if (de) {
        list *list = dictGetVal(de);
        listNode *ln;
        listIter li;

        listRewind(list,&li);
        while ((ln = listNext(&li)) != NULL) {
            client *c = ln->value;

            addReply(c,shared.mbulkhdr[3]);
            addReply(c,shared.messagebulk);
            addReplyBulk(c,channel);
            addReplyBulk(c,message);
            receivers++;
        }
    }
    /* Send to clients listening to matching channels */
    if (listLength(server.pubsub_patterns)) {
        listRewind(server.pubsub_patterns,&li);
        channel = getDecodedObject(channel);
        while ((ln = listNext(&li)) != NULL) {
            pubsubPattern *pat = ln->value;

            if (stringmatchlen((char*)pat->pattern->ptr,
                                sdslen(pat->pattern->ptr),
                                (char*)channel->ptr,
                                sdslen(channel->ptr),0)) {
                addReply(pat->client,shared.mbulkhdr[4]);
                addReply(pat->client,shared.pmessagebulk);
                addReplyBulk(pat->client,pat->pattern);
                addReplyBulk(pat->client,channel);
                addReplyBulk(pat->client,message);
                receivers++;
            }
        }
        decrRefCount(channel);
    }
    return receivers;
}
```
当客户端使用`PUBLISH`向一个通道中发布消息后，该命令会将该消息通知给所有订阅了该通道的客户端，主要包含下列步骤：
- 1、首先取出该频道下的客户端列表。
- 2、遍历该客户端列表，将该消息反馈给每一个客户端。
- 3、除了将消息通知给所有匹配的频道，还会通知给订阅了模式的客户端。

## 总结
- 订阅信息通过`redisServer.pubsub_channels`字典保存，通过通道名称可以获取到所有的订阅客户端列表。
- 模式订阅信息保存在`redisServer.pubsub_patterns`列表中，每一个元素保存了客户端和对应的模式。一个模式可能会与多个频道对应。
- 当有客户端发送一条消息到频道后，服务端会获取对应频道下的所有客户端，并一个一个的发送该条消息。
- 除了订阅频道的客户端可以收到消息，订阅了匹配的模式的客户端也同样可以接收到消息。
