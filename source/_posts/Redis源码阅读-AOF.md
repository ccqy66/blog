---
title: Redis源码阅读-AOF
date: 2018-05-02 19:36:38
tags:
- 源码
- Redis
- AOF
categories:
- AOF
---

## AOF
&emsp;&emsp;通过之前的博客，我们知道，Redis除了是一个内存数据库，其本身还具备持久化的功能。Redis实现了两种持久化方式：RDB和AOF，RDB在之前的博客中已经介绍。本文将详细分析AOF的实现原理。
> 本文不对使用过程进行赘述，如果不了解AOF的使用和特性，可以访问Redis的官方网站了解：https://redis.io/topics/persistence

<!--more-->
AOF为（Append Only File）的缩写，与RDB不同的是，RDB可以看作是当前数据库数据的快照，而AOF则是通过保存Redis服务器所执行的写命令来记录数据库的状态。并且按照增量的模式逐渐的添加到文件中。首先祭出一张图，了解一下AOF工作的整体流程图：
{% asset_img aof_1.png %}

## 原理实现
下面就根据上述的图来分析AOF的实现原理。

### AOF缓冲区
服务端执行的命令并不是实时被写入到文件中，这些命令首先会被放置到内存缓存区中，然后再根据策略（通过配置项fsync来指定）来将内存缓存区的数据写入到文件中。缓冲区由`redisServer`结构体定义：
```C
struct redisServer {
   sds aof_buf;      /* AOF buffer, written before entering the event loop */
}
```
命令进行该缓存区的路径为：processCommand()->call()->propagate()->feedAppendOnlyFile()，现在就让我们看看feedAppendOnlyFile方法，看一下命令是如何添加到该AOF缓存区的：
```C
void feedAppendOnlyFile(struct redisCommand *cmd, int dictid, robj **argv, int argc) {
    //创建一个buf
    sds buf = sdsempty();
    robj *tmpargv[3];

    /* The DB this command was targeting is not the same as the last command
     * we appended. To issue a SELECT command is needed. */
    /**
     * 需要校队当前客户端操作的数据库 与 当前aof操作的数据库
     * 生成SELECT命令，选择数据库
     */
    if (dictid != server.aof_selected_db) {
        char seldb[64];

        snprintf(seldb,sizeof(seldb),"%d",dictid);
        buf = sdscatprintf(buf,"*2\r\n$6\r\nSELECT\r\n$%lu\r\n%s\r\n",
            (unsigned long)strlen(seldb),seldb);
        server.aof_selected_db = dictid;
    }
    /**
     * 翻译命令：
     * 1、将一些命令翻译成精度更高的命令，例如将EXPIRE、PEXPIRE、EXPIREAT翻译成 PEXPIREAT 命令
     * 2、拆分一些命令，例如将SETEX、PSETEX拆分成两个命令：SET和PEXPIREAT
     * 3、与2同义，将SET命令中带有过期时间的参数拆分成两个命令：SET和PEXPIREAT
     */
     /*********省略命令翻译过程***********/
    /* Append to the AOF buffer. This will be flushed on disk just before
     * of re-entering the event loop, so before the client will get a
     * positive reply about the operation performed. */
    /**
     * 如果当前服务器开启了aof，将将当前命令追加到aof缓冲尾部
     *
     */
    if (server.aof_state == AOF_ON)
        server.aof_buf = sdscatlen(server.aof_buf,buf,sdslen(buf));

    /* If a background append only file rewriting is in progress we want to
     * accumulate the differences between the child DB and the current one
     * in a buffer, so that when the child process will do its work we
     * can append the differences to the new append only file. */
    /**
     * 如果当前后台aof重写正在进行中，那么就需要将数据追加到重写缓存区。
     * 父进程会将重写缓存区的数据通过管道的形式写给子进程
     */
    if (server.aof_child_pid != -1)
        aofRewriteBufferAppend((unsigned char*)buf,sdslen(buf));

    sdsfree(buf);
}
```
从上面的代码来看，逻辑还是很简单的：
- 首先根据当前选择的数据库，来校对命令操作的数据库id。
- 将当前指定的命令按照协议翻译成字符串表达形式。
- 将当前命令通过`sdscatlen`函数追加到缓冲区尾部。
- 如果当前正在进行aof重写，还需要把当前命令写入到aof重写缓冲区。

### 文件写入
通过上面的逻辑，当前服务器所执行的所有命令都保存到了内存中，为了能够达到持久化的目的，还需要从内存到文件的处理逻辑。Redis处理这块逻辑是在一个主循环内的`beforeSleep`函数进行的。执行链路为：aeMain()->beforeSleep()->flushAppendOnlyFile()。下面让我们详细的了解`flushAppendOnlyFile`的实现。
```C
void flushAppendOnlyFile(int force) {
    ssize_t nwritten;
    int sync_in_progress = 0;
    mstime_t latency;
    //buf中没有数据就没有必要在写入了
    if (sdslen(server.aof_buf) == 0) return;
    /**
     * 如果当前同步的类型为：每秒同步一次，则需要判断一下当前是否还处于写入文件过程中：sync_in_progress
     */
    if (server.aof_fsync == AOF_FSYNC_EVERYSEC)
        sync_in_progress = bioPendingJobsOfType(BIO_AOF_FSYNC) != 0;

    if (server.aof_fsync == AOF_FSYNC_EVERYSEC && !force) {
        /* With this append fsync policy we do background fsyncing.
         * If the fsync is still in progress we can try to delay
         * the write for a couple of seconds. */
        /**
         * 如果当前还正在处于fysnc过程，就需要delay了
         */
        if (sync_in_progress) {
            /**
             * 如果还没有延期过，就设置当前时间，并直接返回
             */
            if (server.aof_flush_postponed_start == 0) {
                /* No previous write postponing, remember that we are
                 * postponing the flush and return. */
                server.aof_flush_postponed_start = server.unixtime;
                return;
            }
                /**
                 * 如果延期的时间没有超过两秒，就什么也不做
                 */
            else if (server.unixtime - server.aof_flush_postponed_start < 2) {
                /* We were already waiting for fsync to finish, but for less
                 * than two seconds this is still ok. Postpone again. */
                return;
            }
            /**
             * 进入到此处，说明：上一次的aof同步使用了2s还没有完成
             */

            /* Otherwise fall trough, and go write since we can't wait
             * over two seconds. */
            server.aof_delayed_fsync++;
            serverLog(LL_NOTICE,
                      "Asynchronous AOF fsync is taking too long (disk is busy?). Writing the AOF buffer without waiting for fsync to complete, this may slow down Redis.");
        }
    }
    /* We want to perform a single write. This should be guaranteed atomic
     * at least if the filesystem we are writing is a real physical one.
     * While this will save us against the server being killed I don't think
     * there is much to do about the whole server stopping for power problems
     * or alike */

    latencyStartMonitor(latency);
    nwritten = write(server.aof_fd, server.aof_buf, sdslen(server.aof_buf));
    latencyEndMonitor(latency);
    /* We want to capture different events for delayed writes:
     * when the delay happens with a pending fsync, or with a saving child
     * active, and when the above two conditions are missing.
     * We also use an additional event name to save all samples which is
     * useful for graphing / monitoring purposes. */
    if (sync_in_progress) {
        latencyAddSampleIfNeeded("aof-write-pending-fsync", latency);
    } else if (server.aof_child_pid != -1 || server.rdb_child_pid != -1) {
        latencyAddSampleIfNeeded("aof-write-active-child", latency);
    } else {
        latencyAddSampleIfNeeded("aof-write-alone", latency);
    }
    latencyAddSampleIfNeeded("aof-write", latency);

    /* We performed the write so reset the postponed flush sentinel to zero. */
    server.aof_flush_postponed_start = 0;
    /**
     * 疑问：如果再调用write()函数之后，再有请求进来，会导致写入的内容与当前的buf大小不同，会导致失败吗？
     * 解答：不会，因为此函数是在主进程被调用的，在执行该函数的同时是不会接收请求的。
     * 所以可以确认写入量和aof.buf的内容不同时，一定是文件写入时的问题。可能是磁盘空间不够。
     */
    if (nwritten != (signed) sdslen(server.aof_buf)) {
        static time_t last_write_error_log = 0;
        int can_log = 0;

        /* Limit logging rate to 1 line per AOF_WRITE_LOG_ERROR_RATE seconds. */
        if ((server.unixtime - last_write_error_log) > AOF_WRITE_LOG_ERROR_RATE) {
            can_log = 1;
            last_write_error_log = server.unixtime;
        }

        /* Log the AOF write error and record the error code. */
        /**
         * 文件写入失败
         */
        if (nwritten == -1) {
            if (can_log) {
                serverLog(LL_WARNING, "Error writing to the AOF file: %s",
                          strerror(errno));
                server.aof_last_write_errno = errno;
            }
        } else {
            /**
             * 写入成功了，但是写入的内容与buf的内容大小不一致。也就是出现少写的情况
             */
            if (can_log) {
                serverLog(LL_WARNING, "Short write while writing to "
                                  "the AOF file: (nwritten=%lld, "
                                  "expected=%lld)",
                          (long long) nwritten,
                          (long long) sdslen(server.aof_buf));
            }
            /**
             * 将写入的那一部分内容移除了
             */
            if (ftruncate(server.aof_fd, server.aof_current_size) == -1) {
                if (can_log) {
                    serverLog(LL_WARNING, "Could not remove short write "
                            "from the append-only file.  Redis may refuse "
                            "to load the AOF the next time it starts.  "
                            "ftruncate: %s", strerror(errno));
                }
            } else {
                /* If the ftruncate() succeeded we can set nwritten to
                 * -1 since there is no longer partial data into the AOF. */
                nwritten = -1;
            }
            server.aof_last_write_errno = ENOSPC;
        }

        /* Handle the AOF write error. */
        if (server.aof_fsync == AOF_FSYNC_ALWAYS) {
            /* We can't recover when the fsync policy is ALWAYS since the
             * reply for the client is already in the output buffers, and we
             * have the contract with the user that on acknowledged write data
             * is synced on disk. */
            serverLog(LL_WARNING,
                      "Can't recover from AOF write error when the AOF fsync policy is 'always'. Exiting...");
            exit(1);
        } else {
            /* Recover from failed write leaving data into the buffer. However
             * set an error to stop accepting writes as long as the error
             * condition is not cleared. */
            server.aof_last_write_status = C_ERR;
            /**
             * 那就说明清除失败了，只有写入到文件了
             */
            /* Trim the sds buffer if there was a partial write, and there
             * was no way to undo it with ftruncate(2). */
            if (nwritten > 0) {
                server.aof_current_size += nwritten;
                sdsrange(server.aof_buf, nwritten, -1);
            }
            return; /* We'll try again on the next call... */
        }
    } else {
        /* Successful write(2). If AOF was in error state, restore the
         * OK state and log the event. */
        if (server.aof_last_write_status == C_ERR) {
            serverLog(LL_WARNING,
                      "AOF write error looks solved, Redis can write again.");
            server.aof_last_write_status = C_OK;
        }
    }
    server.aof_current_size += nwritten;

    /* Re-use AOF buffer when it is small enough. The maximum comes from the
     * arena size of 4k minus some overhead (but is otherwise arbitrary). */
    /**
     * 如果aof_buf占用的空间很小（<4k），那么就重用这一块内存。
     */
    if ((sdslen(server.aof_buf) + sdsavail(server.aof_buf)) < 4000) {
        sdsclear(server.aof_buf);
    } else {
        sdsfree(server.aof_buf);
        server.aof_buf = sdsempty();
    }

    /* Don't fsync if no-appendfsync-on-rewrite is set to yes and there are
     * children doing I/O in the background. */
    if (server.aof_no_fsync_on_rewrite &&
        (server.aof_child_pid != -1 || server.rdb_child_pid != -1))
        return;

    /* Perform the fsync if needed. */
    /**
     * 如果appendfsync=always时：每次执行时都会执行fsync函数。
     * 如果appendfsync=everysec时：将防止到后台任务中执行。
     */
    if (server.aof_fsync == AOF_FSYNC_ALWAYS) {
        /* aof_fsync is defined as fdatasync() for Linux in order to avoid
         * flushing metadata. */
        latencyStartMonitor(latency);
        aof_fsync(server.aof_fd); /* Let's try to get this data on the disk */
        latencyEndMonitor(latency);
        latencyAddSampleIfNeeded("aof-fsync-always", latency);
        server.aof_last_fsync = server.unixtime;
    } else if ((server.aof_fsync == AOF_FSYNC_EVERYSEC &&
                server.unixtime > server.aof_last_fsync)) {
        if (!sync_in_progress) aof_background_fsync(server.aof_fd);
        server.aof_last_fsync = server.unixtime;
    }
}
```
这一段的内容中，代码比较长，但是流程还是相对简单的，主要的含义都详细的写到上面的注释中，在这里就简单的整理一下逻辑：
- 需要注意一点，调用系统调用`write`，只是将内存中的数据写入到内核的缓存区中，具体何时能够落入磁盘，需要靠操作系统的调度。为了能够强迫操作系统将内核缓存区的内容刷入磁盘，则需要依靠另一个系统调用`fsync`。但是该系统调用是消耗一定的性能，但是可以保证数据的安全性。为了权衡数据安全与性能。Redis提供了一些参数，可以配置调用`fsync`的频率和时机。通过配置appendfsync选项，可以指定同步的频率，主要有三种选项：
 - always：每次命令都需要调用`fsync`系统调用。性能最低，但是数据安全性最高。
 - everysec：每秒种，将aof缓存区的内容通过调用`fsync`系统调用，性能中，数据安全性略低。
 - none：不显示执行`fsync`，将有操作系统调度，性能高，数据安全性很低。
- 当配置appendfsync=everysec时，原则上一次只能同步一个同步任务，如果某一个同步任务时间过长，下一次循环中的同步任务会被延期，不过最多会被延期2秒，2秒后会重新开启一次同步。
- 当配置appendfsync=everysec时，同步任务会被添加到后台任务列表中，由后台线程去处理`fysnc`的系统调用。

### AOF重写
从上面的分析中，我们知道，AOF文件会记录客户端提交的每一个命令，例如，对一个列表的操作，有如下命令：
```C
sadd name chenchen
sadd name cuibq
sadd name liudehua
sadd name zhangxueyou
```
执行如上命令后，此时数据库中的`name`列表的值为：`[chenchen,cuibq,liudehua,zhangxueyou]`，其实完全可以使用一条命令来体现的`sadd name chenchen,cuibq,liudehua,zhangxueyou`，但是在AOF文件中却有4条记录，这样会导致大量的磁盘占用，资源浪费，Redis为了解决此问题，实现了AOF文件重写的功能，就是将多条命令合并成一条或者更少的命令。

#### 实现原理
实现AOF重写功能，Redis并不是读取aof文件，并对aof进行合并操作，而是通过对当前数据库的快照来实现的。实现原理可以用如下图来描述：
{% asset_img rewrite.png %}

当AOF重写时，主进程会fork一个子进程，该子进程会拥有此时主进程数据库的一个快照（实际上就是此时数据库的全量数据），子进程会把快照中的key-value按照数据格式翻译成命令形式，并写入到文件中，在子进程重写的过程中，父进程会正常的进行aof文件的维护，不过多了一个aof重写缓冲区，此时，命令除了会写入到aof缓冲区，同时也会写入到aof重写缓冲区中，aof重写缓冲区的内容会通过管道传递给子进程，然后子进程在把这部分的内容写入到文件。最后把aof重写文件替换成aof文件即可。

#### 代码实现
下面将分模块分析一下重写过程

##### 写入重写缓冲区
在aof文件处理过程中，我们知道`feedAppendOnlyFile`函数中有这么一段：
```C
if (server.aof_child_pid != -1)
        aofRewriteBufferAppend((unsigned char *) buf, sdslen(buf));
```
如果当前后台aof重写正在进行中，那么就需要将数据追加到重写缓存区。

```C
void aofRewriteBufferAppend(unsigned char *s, unsigned long len) {
    //获取最后一个元素
    listNode *ln = listLast(server.aof_rewrite_buf_blocks);
    //判断是否存在元素，如果不存在最后一个元素，实际上就是说当前的重写缓冲区为空
    aofrwblock *block = ln ? ln->value : NULL;

    while (len) {
        /* If we already got at least an allocated block, try appending
         * at least some piece into it. */
        /**
         * 如果当前存在区块
         */
        if (block) {
            //当前区块有多少空间，就填充多少，剩余的部分将重新创建一个新的区块
            unsigned long thislen = (block->free < len) ? block->free : len;
            if (thislen) {  /* The current block is not already full. */
                memcpy(block->buf + block->used, s, thislen);
                block->used += thislen;
                block->free -= thislen;
                s += thislen;
                len -= thislen;
            }
        }
        /**
         * 说明当前区块不够保存该内容，那就需要重新申请一个区块了
         */
        if (len) { /* First block to allocate, or need another block. */
            int numblocks;

            block = zmalloc(sizeof(*block));
            block->free = AOF_RW_BUF_BLOCK_SIZE;
            block->used = 0;
            listAddNodeTail(server.aof_rewrite_buf_blocks, block);

            /* Log every time we cross more 10 or 100 blocks, respectively
             * as a notice or warning. */
            numblocks = listLength(server.aof_rewrite_buf_blocks);
            if (((numblocks + 1) % 10) == 0) {
                int level = ((numblocks + 1) % 100) == 0 ? LL_WARNING :
                            LL_NOTICE;
                serverLog(level, "Background AOF buffer size: %lu MB",
                          aofRewriteBufferSize() / (1024 * 1024));
            }
        }
    }

    /* Install a file event to send data to the rewrite child if there is
     * not one already. */
    if (aeGetFileEvents(server.el, server.aof_pipe_write_data_to_child) == 0) {
        aeCreateFileEvent(server.el, server.aof_pipe_write_data_to_child,
                          AE_WRITABLE, aofChildWriteDiffData, NULL);
    }
}

```
aof重写缓存区并不是像aof缓存区一样，使用`sds`数据结构，而是通过使用`aofrwblock`的数据结构：
```C
typedef struct aofrwblock {
    unsigned long used, free;
    char buf[AOF_RW_BUF_BLOCK_SIZE];
} aofrwblock;
```
默认情况一下一个区块占用10M的内存空间，并且每一个区块之前通过列表来组织的。

##### 进程间通信
由于aof重写缓冲区是处于主进程中的，该缓存区对子进程而言是不可见的，所以为了达到主进程和子进程之间进行数据通信，采取管道的形式。在aof重写过程中，使用了多个管道：
```C
   /**
    * 父进程写数据到子进程的管道
    */
   server.aof_pipe_write_data_to_child = fds[1];
   /**
    * 从父进程读数据的管道
    */
   server.aof_pipe_read_data_from_parent = fds[0];
   /**
    * 子进程写ack给父进程
    */
   server.aof_pipe_write_ack_to_parent = fds[3];
   /**
    * 父进程读ack从子进程
    */
   server.aof_pipe_read_ack_from_child = fds[2];
   /**
    * 父进程写ack给子进程
    */
   server.aof_pipe_write_ack_to_child = fds[5];
   /**
    * 子进程读ack从父进程
    */
   server.aof_pipe_read_ack_from_parent = fds[4];
   /**
    * aof停止发送diff数据
    */
   server.aof_stop_sending_diff = 0;
```
其中主进程将aof重写缓存区的数据传递给子进程通过管道`aof_pipe_write_data_to_child`，将主进程aof重写缓存区的内容通过管道写给子进程通过`aofChildWriteDiffData`函数完成的：
```C
void aofChildWriteDiffData(aeEventLoop *el, int fd, void *privdata, int mask) {
    listNode *ln;
    aofrwblock *block;
    ssize_t nwritten;
    UNUSED(el);
    UNUSED(fd);
    UNUSED(privdata);
    UNUSED(mask);

    while (1) {
        ln = listFirst(server.aof_rewrite_buf_blocks);
        block = ln ? ln->value : NULL;
        if (server.aof_stop_sending_diff || !block) {
            aeDeleteFileEvent(server.el, server.aof_pipe_write_data_to_child,
                              AE_WRITABLE);
            return;
        }
        if (block->used > 0) {
            /**
             * 将重写缓存区的数据写入通道
             */
            nwritten = write(server.aof_pipe_write_data_to_child,
                             block->buf, block->used);
            if (nwritten <= 0) return;
            memmove(block->buf, block->buf + nwritten, block->used - nwritten);
            block->used -= nwritten;
            block->free += nwritten;
        }
        if (block->used == 0) listDelNode(server.aof_rewrite_buf_blocks, ln);
    }
}
```
这一部分的逻辑很简单，循环读取aof重写缓存区的内容，并将内容写到管道`aof_pipe_write_data_to_child`中，子进程可以通过管道`aof_pipe_read_data_from_parent`读取数据。

##### aof文件重写
aof重写文件的数据来源有两个，一个是当前的数据快照，另一个就是aof重写缓冲区的内容。
```C
int rewriteAppendOnlyFile(char *filename) {
    rio aof;
    FILE *fp;
    char tmpfile[256];
    char byte;
    /**
     * 创建临时文件，该文件会在aof重写缓冲区写入时也会使用的
     */
    /* Note that we have to use a different temp name here compared to the
     * one used by rewriteAppendOnlyFileBackground() function. */
    snprintf(tmpfile, 256, "temp-rewriteaof-%d.aof", (int) getpid());
    //打开临时文件
    fp = fopen(tmpfile, "w");
    if (!fp) {
        serverLog(LL_WARNING, "Opening the temp file for AOF rewrite in rewriteAppendOnlyFile(): %s", strerror(errno));
        return C_ERR;
    }

    server.aof_child_diff = sdsempty();
    rioInitWithFile(&aof, fp);

    if (server.aof_rewrite_incremental_fsync)
        rioSetAutoSync(&aof, AOF_AUTOSYNC_BYTES);
    //是否开启了aof、rdb混合持久，如果开启了，就首先持久化rdb文件
    if (server.aof_use_rdb_preamble) {
        int error;
        if (rdbSaveRio(&aof, &error, RDB_SAVE_AOF_PREAMBLE, NULL) == C_ERR) {
            errno = error;
            goto werr;
        }
    } else {
        if (rewriteAppendOnlyFileRio(&aof) == C_ERR) goto werr;
    }

    /* Do an initial slow fsync here while the parent is still sending
     * data, in order to make the next final fsync faster. */
    /**
     * 将数据刷到磁盘
     */
    if (fflush(fp) == EOF) goto werr;
    if (fsync(fileno(fp)) == -1) goto werr;

    /* Read again a few times to get more data from the parent.
     * We can't read forever (the server may receive data from clients
     * faster than it is able to send data to the child), so we try to read
     * some more data in a loop as soon as there is a good chance more data
     * will come. If it looks like we are wasting time, we abort (this
     * happens after 20 ms without new data). */
    /**
     * 当前的数据库的内容写入到文件之后，就需要处理在重写过程中的重写缓冲区的数据了。
     * 这部分数据会通过读取主进程的通道。
     * 如果在20ms内没有新的数据产生的话，就终止了。
     */
    int nodata = 0;
    mstime_t start = mstime();
    while (mstime() - start < 1000 && nodata < 20) {
        if (aeWait(server.aof_pipe_read_data_from_parent, AE_READABLE, 1) <= 0) {
            nodata++;
            continue;
        }
        nodata = 0; /* Start counting from zero, we stop on N *contiguous*
                       timeouts. */
        //将主进程通过管道发送过来的数据存储到server.aof_child_diff中
        aofReadDiffFromParent();
    }

    /* Ask the master to stop sending diffs. */
    /**
     * 告诉主进程，不要在发送命令到重写缓冲区了
     */
    if (write(server.aof_pipe_write_ack_to_parent, "!", 1) != 1) goto werr;
    if (anetNonBlock(NULL, server.aof_pipe_read_ack_from_parent) != ANET_OK)
        goto werr;
    /* We read the ACK from the server using a 10 seconds timeout. Normally
     * it should reply ASAP, but just in case we lose its reply, we are sure
     * the child will eventually get terminated. */
    if (syncRead(server.aof_pipe_read_ack_from_parent, &byte, 1, 5000) != 1 ||
        byte != '!')
        goto werr;
    serverLog(LL_NOTICE, "Parent agreed to stop sending diffs. Finalizing AOF...");

    /* Read the final diff if any. */
    //读取在发送确认信息过程中的命令
    aofReadDiffFromParent();

    /* Write the received diff to the file. */
    serverLog(LL_NOTICE,
              "Concatenating %.2f MB of AOF diff received from parent.",
              (double) sdslen(server.aof_child_diff) / (1024 * 1024));
    //把diff数据写入到新aof文件中
    if (rioWrite(&aof, server.aof_child_diff, sdslen(server.aof_child_diff)) == 0)
        goto werr;
    /**
     * 确保数据落入磁盘
     * fflush==>将内存数据输入到操作系统的内核缓存区
     * fsync==>将内核缓冲区的数据持久化到磁盘中
     */
    /* Make sure data will not remain on the OS's output buffers */
    if (fflush(fp) == EOF) goto werr;
    if (fsync(fileno(fp)) == -1) goto werr;
    if (fclose(fp) == EOF) goto werr;

    /* Use RENAME to make sure the DB file is changed atomically only
     * if the generate DB file is ok. */
    /**
     * 将临时文件重命名成配置的aof文件名
     * 由于rename系统调用是原子操作。所以不用担心在rename过程中会有新的命令丢失。
     */
    if (rename(tmpfile, filename) == -1) {
        serverLog(LL_WARNING, "Error moving temp append only file on the final destination: %s", strerror(errno));
        unlink(tmpfile);
        return C_ERR;
    }
    serverLog(LL_NOTICE, "SYNC append only file rewrite performed");
    return C_OK;

    werr:
    serverLog(LL_WARNING, "Write error writing append only file on disk: %s", strerror(errno));
    fclose(fp);
    unlink(tmpfile);
    return C_ERR;
}
```
这个函数是aof重写过程中，涉及的过程比较多，稍微复杂一点的地方。下面详细分析一下过程：
- 写入文件之前，会首先创建一个临时文件，文件的命名格式为：`temp-rewriteaof-%d.aof`。
- 将当前数据库中的快照写入到这个临时文件中。
- 循环读取主进程通过管道同步过来的数据，并将数据写入到`aof_child_diff`结构中，保存。如果再1s内，有20ms的间隔没有数据同步过来，就停止本次同步了。
- 通过管道`aof_pipe_write_ack_to_parent`通知主进程，别在往aof重写缓存区写数据了，通知的方式是，向通道中写入一个标识:`!`。
```C
 if (write(server.aof_pipe_write_ack_to_parent, "!", 1) != 1) goto werr;
```
- 主进程接收到这个标识后，就停止向aof重写缓存区写命令了。
- 由于在确认过程中，父进程还会有一小段时间向aof缓冲区写数据的，所以还需要读取这一小段时间的命令。
- 将aof_child_diff的数据全部写入到文件中。
- 重命名临时文件为配置的aof正式文件。
- 至此，一次aof重写完成。


## 总结
- AOF文件与RDB文件不同的是：
 - AOF:保存的是服务端执行的命令。
 - RDB:保存的是当前服务器数据库的全部键值。
- 命令不会实时的写入到文件，会首先保存到缓存区中，再通过策略写入到文件。
- AOF重写的目的是降低AOF文件的大小，提高AOF文件的利用率。AOF的实现并不是读取AOF文件，对文件的合并。而是通过对数据库快照的操作。
- 通过调用`write`和`flush`系统调用，是没有办法实时将数据写入磁盘的，为了能够确保写入到磁盘，还需要调用`fsync`系统调用。
