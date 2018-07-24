---
title: Redis源码阅读-RDB机制
date: 2018-03-11 14:26:27
tags:
- 源码
- Redis
- RDB
categories:
- REDIS
---
## 简介
&emsp;&emsp;众所周知，Redis是一个内存数据库，常用于缓存存储，但是Redis也有一个特性，就是可以进行持久化操作，Redis支持两种持久化方式：AOF和RDB。本文将主要讲解RDB的原理实现。
> 关于Redis的持久化详细资料。本文不做赘述<br>可以查看官方文档：https://redis.io/topics/persistence <br/>或查看中文翻译版：http://www.redis.cn/topics/persistence.html

<!--more-->
## 文件格式
&emsp;&emsp;在正常情况下，Redis的数据都是存储在内存中，为了让这些数据在Redis重启之后仍然可用，Redis需要定期将数据保存到文件中，RDB就是一种定期将内存数据库中的数据保存到文件（dump.rdb）的一个技术。当Redis重启时，通过加载RDB文件的数据来还原数据库的状态。为了系统在重启时能够正确加载写入到文件的数据，那么就必须定义一种文件写入的格式，以保证正确的加载。
### 文件布局
```
-------------------------------------
|文件标识|辅助信息|数据库数据|结束符|校验和|
-------------------------------------
```
#### 文件标识
---
```
----------------
|REDIS|文件版本号|
----------------
```
例如：REDIS0007
#### 辅助信息
---
```
----------------------------------
|FA redis-ver len value
|FA redis-bits value
|FA ctime value
|FA used-mem value
----------------------------------
```
其中：FA代表该行数据类型为辅助信息。
例如：

```
FA redis-ver 005 3.2.6
FA redis-bits 64
FA ctime 1520760140
FA used-mem 100766417207
```
#### 数据库数据
---
```
----------------------------------
FE index
FB db_size expired_size
{
FC expired_time value_type key_len key value_len value
or
value_type key_len key value_len value
}
...
FF
----------------------------------
```
>对于key-value数据的存储，有两种形式：如果对应key有过期时间的话就写入`FC expired_time value_type key_len key value_len value`，如果没有过期时间的话就写入`value_type key_len key value_len value`

例如：
```
----------------------------------
FE 00
FB 1 0
{
FC 1520760140 OBJ_STRING 8 username 8 chenchen
or
OBJ_STRING 8 username 8 chenchen
}
...
FF
----------------------------------
```
#### 校验和
---
```
----------------
|check_sum
----------------
```

## 文件的读写

### 文件写入
---
文件的写入有两种触发形式：
- 1：手动执行`SAVE`或`BGSAVE`命令。
- 2：由Redis的后台任务定时执行，不过写入的流程也与1中的方式相同，不同的是由Redis定时按照某一条件执行的

下面将分别介绍这两种方式的执行流程。
#### 手动执行：
&emsp;&emsp;首先我们先进入`SAVE`命令的入口：`rdb.c/saveCommand`
```C
void saveCommand(client *c) {
    if (server.rdb_child_pid != -1) { //1
        addReplyError(c,"Background save already in progress");
        return;
    }
    if (rdbSave(server.rdb_filename) == C_OK) {//2
        addReply(c,shared.ok);
    } else {
        addReply(c,shared.err);
    }
}
```
- 1、首先判断此时是否还正在执行`BGSAVE`命令，由于执行`BGSAVE`命令是在子进程中执行的，所以只需要判断是否存在子进程即可。如果正在执行，此时就不会在执行`SAVE`命令了，因为这两条命令的功能是重合的。
- 2、执行保存逻辑，也是真正保存逻辑的入口。

下面我们将进入主流程：`rdb.c/rdbSave`

```C
/* 将数据库的信息保存到磁盘，如果失败就返回C_ERR，成功就返回C_OK */
int rdbSave(char *filename) {
    char tmpfile[256];
    char cwd[MAXPATHLEN]; /* Current working dir path for error messages. */
    FILE *fp;
    rio rdb;
    int error = 0;
    /* 初始化临时文件名称 */
    snprintf(tmpfile,256,"temp-%d.rdb", (int) getpid());
    /* 打开临时文件 */
    fp = fopen(tmpfile,"w");
    if (!fp) {
        //获取当前的工作目录
        char *cwdp = getcwd(cwd,MAXPATHLEN);
        //将错误信息打印到dump.rdb文件中
        serverLog(LL_WARNING,
            "Failed opening the RDB file %s (in server root dir %s) "
            "for saving: %s",
            filename,
            cwdp ? cwdp : "unknown",
            strerror(errno));
        return C_ERR;
    }
    //将文件流信息初始化到rio中，rio变量将伴随整个写入过程
    rioInitWithFile(&rdb,fp);
    //执行写入
    if (rdbSaveRio(&rdb,&error) == C_ERR) {
        errno = error;
        goto werr;
    }

    /* Make sure data will not remain on the OS's output buffers */
    //将缓冲区的内容写入到文件中
    if (fflush(fp) == EOF) goto werr;
    if (fsync(fileno(fp)) == -1) goto werr;
    if (fclose(fp) == EOF) goto werr;

    /* Use RENAME to make sure the DB file is changed atomically only
     * if the generate DB file is ok. */
    //将临时文件重命名成dump.rdb
    //临时文件的名称为：temp-${pid}.rdb
    if (rename(tmpfile,filename) == -1) {
        char *cwdp = getcwd(cwd,MAXPATHLEN);
        serverLog(LL_WARNING,
            "Error moving temp DB file %s on the final "
            "destination %s (in server root dir %s): %s",
            tmpfile,
            filename,
            cwdp ? cwdp : "unknown",
            strerror(errno));
        unlink(tmpfile);
        return C_ERR;
    }

    serverLog(LL_NOTICE,"DB saved on disk");
    //重置dirty。dirty记录了从上一次写入文件档本次写入文件期间执行的数据变更操作数
    server.dirty = 0;
    //记录lastsave时间（本次执行完成时间），用于下次写入文件时判断条件。
    server.lastsave = time(NULL);
    server.lastbgsave_status = C_OK;
    return C_OK;

werr:
    serverLog(LL_WARNING,"Write error saving DB on disk: %s", strerror(errno));
    fclose(fp);
    unlink(tmpfile);
    return C_ERR;
}
```
NOTE:上面的代码虽然很长，但是功能还是挺简单的，总结一下有以下几个步骤：
- 1、创建一个临时文件。写入时会先写入到这个临时文件中。
- 2、初始化文件句柄到rio中。
- 3、按照协议写入数据库信息。
- 4、重命名临时文件为dump.rdb

此后将按照文件定义的格式分块写入到文件，下面将按照文件的布局进行代码分析：`rdb.c/rdbSaveRio`
开始之前我们先了解几个函数：
```
/*该函数相当于包装了系统函数fwrite()：将p指针len长度的内容写入到文件*/
static int rdbWriteRaw(rio *rdb, void *p, size_t len)
/*写入数据类型，一般表示后面所跟数据的类型，例如:FE 00 中FE就是类型，代表着数据库选择*/
int rdbSaveType(rio *rdb, unsigned char type)
/*
 * 将字符串写入到磁盘中，写入到文件有几种表现形式
 * -------------------------------------------------------------------
 * |       字符串类型        |     保存到文件中的形式  |          例子     |
 * -------------------------------------------------------------------
 * |      可以转换成数字      |  将转换成的数字写入到磁盘|      "1234"=>123 |
 * -------------------------------------------------------------------
 * | 开启LZF压缩，且长度大于20 |        使用LZF压缩    |                  |
 * -------------------------------------------------------------------
 * |          其他情况       |     保存长度和字符串    | "1234"=> 4 "1234"|
 * -------------------------------------------------------------------
 */
ssize_t rdbSaveRawString(rio *rdb, unsigned char *s, size_t len)
```
##### 文件标识写入：
```C
//生成文件表示信息。例如：REDIS0007
snprintf(magic,sizeof(magic),"REDIS%04d",RDB_VERSION);
//将文件信息写入到文件
if (rdbWriteRaw(rdb,magic,9) == -1) goto werr;
```
##### 辅助信息写入：
```C
//写入内容：FD 9 redis-ver 5 3.3.6
if (rdbSaveAuxFieldStrStr(rdb,"redis-ver",REDIS_VERSION) == -1) return -1;
//写入内容：FD 10 redis-bits 64
if (rdbSaveAuxFieldStrInt(rdb,"redis-bits",redis_bits) == -1) return -1;
//写入内容：FD 5 ctime 1520760140
if (rdbSaveAuxFieldStrInt(rdb,"ctime",time(NULL)) == -1) return -1;
//写入内容：FD 8 used-mem 100766417207
if (rdbSaveAuxFieldStrInt(rdb,"used-mem",zmalloc_used_memory()) == -1) return -1;

/*将key和value都以string的形式写入到文件*/
int rdbSaveAuxField(rio *rdb, void *key, size_t keylen, void *val, size_t vallen) {
    if (rdbSaveType(rdb,RDB_OPCODE_AUX) == -1) return -1;
    if (rdbSaveRawString(rdb,key,keylen) == -1) return -1;
    if (rdbSaveRawString(rdb,val,vallen) == -1) return -1;
    return 1;
}
```

##### 数据库信息写入：
```C
for (j = 0; j < server.dbnum; j++) {
       .....
       /* 写入数据库的选择器 例如第一个数据库 FE 00 */
       if (rdbSaveType(rdb,RDB_OPCODE_SELECTDB) == -1) goto werr; //1.1
       if (rdbSaveLen(rdb,j) == -1) goto werr;                    //1.2

       uint32_t db_size, expires_size;
       /*当前数据库的键值大小*/
       db_size = (dictSize(db->dict) <= UINT32_MAX) ?
                               dictSize(db->dict) :
                               UINT32_MAX;
       /*过期的键值大小*/
       expires_size = (dictSize(db->expires) <= UINT32_MAX) ?
                               dictSize(db->expires) :
                               UINT32_MAX;

       if (rdbSaveType(rdb,RDB_OPCODE_RESIZEDB) == -1) goto werr;//2.1
       if (rdbSaveLen(rdb,db_size) == -1) goto werr;             //2.2
       if (rdbSaveLen(rdb,expires_size) == -1) goto werr;        //2.3

       /* Iterate this DB writing every entry */
       while((de = dictNext(di)) != NULL) {
           sds keystr = dictGetKey(de);
           robj key, *o = dictGetVal(de);
           long long expire;

           initStaticStringObject(key,keystr);
           expire = getExpire(db,&key);
           if (rdbSaveKeyValuePair(rdb,&key,o,expire,now) == -1) goto werr;//3
       }
       dictReleaseIterator(di);
       .....
       if (rdbSaveType(rdb,RDB_OPCODE_EOF) == -1) goto werr;//4
   }
/*保存单个key value到文件*/
int rdbSaveKeyValuePair(rio *rdb, robj *key, robj *val,
                        long long expiretime, long long now)
{
    /* Save the expire time */
    if (expiretime != -1) {
        /* If this key is already expired skip it */
        if (expiretime < now) return 0;
        if (rdbSaveType(rdb,RDB_OPCODE_EXPIRETIME_MS) == -1) return -1;//3.1
        if (rdbSaveMillisecondTime(rdb,expiretime) == -1) return -1;//3.2
    }

    /* Save type, key, value */
    if (rdbSaveObjectType(rdb,val) == -1) return -1;//3.3
    if (rdbSaveStringObject(rdb,key) == -1) return -1;//3.4
    if (rdbSaveObject(rdb,val) == -1) return -1;//3.5
    return 1;
}
```
- 1：写入 数据库选择器数据（表示当前是第几个数据库，redis初始化会创建16个数据库）
 - 1.1：写入数据库选择器类型：`FE`
 - 1.2：写入数据库的索引`00`(表示是第一个数据库)
- 2：写入 数据库的信息，包含当前数据库的键值大小和过期键值数。
 - 2.1：写入数据库信息类型：`FB`
 - 2.2：写入数据库键值大小：`100`(表示一共有100个键值对)
 - 2.3：写入数据库过期键值：`10`(表示一共有10个键值过期)
- 3：写入键值对。
 - 3.1：如果有过期时间，写入过期时间类型：`FC`
 - 3.2：如果有过期时间，写入过期时间：`1520760140`
 - 3.3：写入value的类型：例如：`0`表示字符串
 - 3.4：写入key的数据：例如：`5 cuibq`
 - 3.5：写入value数据：例如：`cuibq`
- 4：写入数据库结束类型：`FF`

##### 写入校验和
```
if (rioWrite(rdb,&cksum,8) == 0) goto werr;
```

#### 自动写入
---
&emsp;&emsp;自动写入是redis后台进程有定时任务执行的，每次执行时会判断两个参数：保存间隔 和 两次保存期间变动的键值数。这两个参数是由Redis配置文件指定的：
```
save <seconds> <changes>
例如：save 900 1 表示：900秒以后，如果有至少一个键值被改变就会执行保存
```
&emsp;&emsp;我们直接分析自动写入的逻辑(serverCron()函数是redis的一个定时执行任务，默认每隔100ms会执行一次，比较复杂，暂时只分析与rdb文件写入相关的代码)`server.c/serverCron`
```
if (server.dirty >= sp->changes &&
    server.unixtime-server.lastsave > sp->seconds &&
    (server.unixtime-server.lastbgsave_try > CONFIG_BGSAVE_RETRY_DELAY
       ||server.lastbgsave_status == C_OK)
     )
   {
      serverLog(LL_NOTICE,"%d changes in %d seconds. Saving...",sp->changes, (int)sp->seconds);
                rdbSaveBackground(server.rdb_filename);
                break;
  }
```
在分析上述代码之前先讲几个属性：
```
server.dirty：记录从上次文件写入到此时，redis一共进行多少次键值变更操作（增加、删除、修改）
server.lastbgsave_try：上次执行bgsave命令的时间
server.lastbgsave_status：上次执行bgsave命令的状态：成功或失败

```
所以要执行后台的文件写入，必须当且仅当满足以下条件：
- 当前距离上次执行`SAVE`命令时间必须大于配置的时间间隔 即命令中的`<seconds>`参数
- 当前距离上次写入文件操作时间内键值的改变数量必须大于等于配置的数量 即命令中的`<changes>`参数
- 上次执行`BGSAVE`命令成功 或者 虽然失败，但是已经相隔了CONFIG_BGSAVE_RETRY_DELAY时间

## 文件分析
首先我们首先把数据库中的所有数据全部删除，只插入一条数据：
```
> flushdb
> set cuibq cuibq
```
然后我们开始分析，生成的dump.rdb文件：

```
od -c dump.rdb
0000000    R   E   D   I   S   0   0   0   7 372  \t   r   e   d   i   s
0000020    -   v   e   r 005   3   .   2   .   6 372  \n   r   e   d   i
0000040    s   -   b   i   t   s 300   @ 372 005   c   t   i   m   e 136
0000060    ^   ˨  **   Z 372  \b   u   s   e   d   -   m   e   m 302    
0000100    a 017  \0 376  \0 373 001  \0  \0 005   c   u   i   b   q 005
0000120    c   u   i   b   q 377 320 033   q   K 230   Ղ  **   S  
```
同时获取对应的八进制表示，目的是为了解释一些乱码问题：
```
ob -b dump.rdb
0000000   122 105 104 111 123 060 060 060 067 372 011 162 145 144 151 163
0000020   055 166 145 162 005 063 056 062 056 066 372 012 162 145 144 151
0000040   163 055 142 151 164 163 300 100 372 005 143 164 151 155 145 302
0000060   136 313 250 132 372 010 165 163 145 144 055 155 145 155 302 040
0000100   141 017 000 376 000 373 001 000 000 005 143 165 151 142 161 005
0000120   143 165 151 142 161 377 320 033 161 113 230 325 202 123
```
下面按照模块进行分析每一块的内容：
```
R   E   D   I   S   0   0   0   7
```
该部分的内容为"REDIS0007"对应于文件信息。

```
372  \t   r   e   d   i   s -   v   e   r 005   3   .   2   .   6
```
- 372：表示是一个辅助信息的开始。
- \t redis-ver ：表示的内容为：“9 redis-ver”，因为是字符串且长度<20，所以会按照格式“len value”的形式保存到文件中。
- 005 3.2.6：表示的内容为：“5 3.2.6”，同上解释。

```
372  \n   r   e   d   i   s   -   b   i   t   s 300   @
```
- 372：表示是一个辅助信息的开始。
- \n redis-bits：表示的内容为：“10 redis-bits”,解释同上。
- 300   @ ：实际上有效部分是64，@是64对应的ASCII码展示

```
376  \0 373 001  \0  \0 005   c   u   i   b   q 005   c   u   i   b   q 377
```
- 376 \0：表示的内容为“FE 00”,表示数据库选择器开始，且当前操作第一个数据库。
- 373 001  \0：表示的内容为“FB 1 0”，表示当前数据库中只有一个键值对，且没有一个过期键值。
- \0 005   c   u   i   b   q 005   c   u   i   b   q：表示的内容为：“0 5 cuibq 5 cuibq”,表示当前value的类型为string，key有5个长度，为“cuibq”，value也有5个长度为“cuibq”
- 377 ：表示的内容为："FF"，表示数据库操作完成。

```
320 033   q   K 230   Ղ  **   S
320 033 161 113 230 325 202 123
```
这最后的八个字节表示的就是校验和
