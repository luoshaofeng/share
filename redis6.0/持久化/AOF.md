# Info

版本信息：Redis 6.0



# AOF

aof文件格式有两种：`纯aof格式`文件和`rdb与aof混合格式`文件



## 加载

当redis服务器启动时，如果开启aof持久化，那么就会读取aof文件，解析aof文件，把数据写入到redis数据库中

如果是混合模式，分成两部分：

- rdb格式部分：针对rdb部分，redis直接解析出key和val，然后添加到数据库中
- aof格式部分：针对aof部分，会构造出一个fakeClient，模拟redis服务器接收客户端的行为，执行命令。
  - aof保存的数据与客户端发生到服务器接收到的数据是完全一致的，刚好借助client就能完成命令的执行

```c
void loadDataFromDisk(void) {
    long long start = ustime();
    // 如果aof是开启的
    if (server.aof_state == AOF_ON) {
        // 加载aof文件
        if (loadAppendOnlyFile(server.aof_filename) == C_OK)
            serverLog(LL_NOTICE, "DB loaded from append only file: %.3f seconds", (float) (ustime() - start) / 1000000);
    } else {        // aof没开就加载rdb文件
        ……
        if (rdbLoad(server.rdb_filename, &rsi,RDBFLAGS_NONE) == C_OK) {
            ……
        } else if (errno != ENOENT) {
            serverLog(LL_WARNING, "Fatal error loading the DB: %s. Exiting.", strerror(errno));
            exit(1);
        }
    }
}

```

从aof文件中将数据解析到数据库，aof格式的文件如

```tex
*3\r\n
$3\r\n
SET\r\n
$3\r\n
key\r\n
$5\r\n
value\r\n
```

解析代码：

```c
int loadAppendOnlyFile(char *filename) {
  	// 模拟处理客户端请求
    struct client *fakeClient;
    // 打开aof文件
    FILE *fp = fopen(filename, "r");
    ……

    if (fp && redis_fstat(fileno(fp), &sb) != -1 && sb.st_size == 0) {
        server.aof_current_size = 0;
        server.aof_fsync_offset = server.aof_current_size;
        fclose(fp);
        return C_ERR;
    }

    // 暂时关闭aof
    server.aof_state = AOF_OFF;

    //创建aof客户端
    fakeClient = createAOFClient();
    ……
      
    char sig[5]; /* "REDIS" */
    // 查看文件头是不是"REDIS"开头，是的话就是rdb和aof混合模式
    if (fread(sig, 1, 5, fp) != 5 || memcmp(sig, "REDIS", 5) != 0) {
        /* No RDB preamble, seek back at 0 offset. */
        // 不是混合模式，指针重新指向文件头
        if (fseek(fp, 0,SEEK_SET) == -1) goto readerr;
    } else {
       	// 混合模式
        rio rdb;

        // 指针重新指向文件头
        if (fseek(fp, 0,SEEK_SET) == -1) goto readerr;
        // 初始化rdb io
        rioInitWithFile(&rdb, fp);
        // 加载文件中rdb格式的数据
        if (rdbLoadRio(&rdb,RDBFLAGS_AOF_PREAMBLE,NULL) != C_OK) {
            serverLog(LL_WARNING, "Error reading the RDB preamble of the AOF file, AOF loading aborted");
            goto readerr;
        } else {
            serverLog(LL_NOTICE, "Reading the remaining AOF tail...");
        }
    }

    // 加载文件中aof格式的数据
    while (1) {
        int argc, j;
        unsigned long len;
        robj **argv;
        char buf[128];
        sds argsds;
        struct redisCommand *cmd;

        ……

        // 读取一行数据
        if (fgets(buf, sizeof(buf), fp) == NULL) {
            if (feof(fp))
                break;
            else
                goto readerr;
        }
        if (buf[0] != '*') goto fmterr;
        if (buf[1] == '\0') goto readerr;
      
        // 解析命令个数
        argc = atoi(buf + 1);
        if (argc < 1) goto fmterr;

        // 分配argc个对象
        argv = zmalloc(sizeof(robj *) * argc);
        fakeClient->argc = argc;
        fakeClient->argv = argv;

      	// 针对命令个数解析出命令
        for (j = 0; j < argc; j++) {
            /* Parse the argument len. */
            char *readres = fgets(buf, sizeof(buf), fp);
            if (readres == NULL || buf[0] != '$') {
                fakeClient->argc = j; /* Free up to j-1. */
                freeFakeClientArgv(fakeClient);
                if (readres == NULL)
                    goto readerr;
                else
                    goto fmterr;
            }
            len = strtol(buf + 1,NULL, 10);

            // 构造string对象
            argsds = sdsnewlen(SDS_NOINIT, len);
            if (len && fread(argsds, len, 1, fp) == 0) {
                sdsfree(argsds);
                fakeClient->argc = j; /* Free up to j-1. */
                freeFakeClientArgv(fakeClient);
                goto readerr;
            }
           
          	// string对象转成object
            // 解析出命令，解析出来的命令放到client中。处理客户端命令也是这么做的
            argv[j] = createObject(OBJ_STRING, argsds);

            /* Discard CRLF. */
            if (fread(buf, 2, 1, fp) == 0) {
                fakeClient->argc = j + 1; /* Free up to j. */
                freeFakeClientArgv(fakeClient);
                goto readerr;
            }
        }

        // 相当于伪造出一个client
        /* Command lookup */
        cmd = lookupCommand(argv[0]->ptr);
        if (!cmd) {
            serverLog(LL_WARNING,
                      "Unknown command '%s' reading the append only file",
                      (char *) argv[0]->ptr);
            exit(1);
        }

      	// 事务
        if (cmd == server.multiCommand) valid_before_multi = valid_up_to;

        /* Run the command in the context of a fake client */
        // 设置cmd
        fakeClient->cmd = fakeClient->lastcmd = cmd;
        // 事务的话就执行事务
        if (fakeClient->flags & CLIENT_MULTI &&
            fakeClient->cmd->proc != execCommand) {
            queueMultiCommand(fakeClient);
        } else {
            // 执行命令，就像是客户端发送的命令
            cmd->proc(fakeClient);
        }

        ……
    }

    // 不完整事务，退回。最后一部分，事务可能保存不完整（异常），要处理一下
    if (fakeClient->flags & CLIENT_MULTI) {
        serverLog(LL_WARNING,
                  "Revert incomplete MULTI/EXEC transaction in AOF file");
        valid_up_to = valid_before_multi;
        goto uxeof;
    }

loaded_ok: /* DB loaded, cleanup and return C_OK to the caller. */
    fclose(fp);
    freeFakeClient(fakeClient);
    server.aof_state = old_aof_state;
    stopLoading(1);
    aofUpdateCurrentSize();
    server.aof_rewrite_base_size = server.aof_current_size;
    server.aof_fsync_offset = server.aof_current_size;
    return C_OK;

readerr: /* Read error. If feof(fp) is true, fall through to unexpected EOF. */
    ……

uxeof: /* Unexpected AOF end of file. */
    ……
}
```





## 文件重写条件

aof文件重写会fork出一个子进程，不影响主进程的执行。



当文件过大时，会进行重写

重写的条件（两个条件同时满足）：

- 百分比：当此次aof文件的大小比上次重写的文件大小增长了百分之N
- 文件大小阈值：当此次aof文件的大小超过了阈值

这两项都是配置项。



redis有一个周期执行函数`serverCron`，会检测需不需要重写aof文件

```c
int serverCron(struct aeEventLoop *eventLoop, long long id, void *clientData) {
    ……

    
    if (hasActiveChildProcess() || ldbPendingChildren()) {
        checkChildrenDone();
    } else {
      
        // RDB
        for (j = 0; j < server.saveparamslen; j++) {
            ……
        }

        // 重写aof文件的条件
        if (server.aof_state == AOF_ON &&
            !hasActiveChildProcess() &&			// 没有子进程在活动
            server.aof_rewrite_perc &&      // 百分比阈值
            server.aof_current_size > server.aof_rewrite_min_size) {        // 当前文件大小大于阈值
            // 获取上一次完成aof文件重写时的大小
            long long base = server.aof_rewrite_base_size ? server.aof_rewrite_base_size : 1;
            // 计算这次文件与上次文件增长的百分比
            long long growth = (server.aof_current_size * 100 / base) - 100;
            // 大于等于百分比，触发重写
            if (growth >= server.aof_rewrite_perc) {
                serverLog(LL_NOTICE, "Starting automatic rewriting of AOF on %lld%% growth", growth);
              	// 重写aof文件
                rewriteAppendOnlyFileBackground();
            }
        }
    }
    
  	……	
  
    return 1000 / server.hz;
}
```



# AOF文件重写

redis会创建一个子进程来进行aof重写

- 创建管道`aofCreatePipes`：创建3对管道，用来同步命令和同步信息
- 创建子进程信息管道：用于主进程接收子进程的一些信息
- fork子进程

```c
int rewriteAppendOnlyFileBackground(void) {
    pid_t childpid;

    // 有活跃的子进程，返回错误
    if (hasActiveChildProcess()) return C_ERR;
    // 创建aof管道
    if (aofCreatePipes() != C_OK) return C_ERR;
    // 打开子进程信息管道
    openChildInfoPipe();
    if ((childpid = redisFork(CHILD_TYPE_AOF)) == 0) {      // 子进程
        char tmpfile[256];

        /* Child */
        // 设置进程名
        redisSetProcTitle("redis-aof-rewrite");
        // 设置cpu亲和性
        redisSetCpuAffinity(server.aof_rewrite_cpulist);
        // 初始化文件名
        snprintf(tmpfile, 256, "temp-rewriteaof-bg-%d.aof", (int) getpid());
        if (rewriteAppendOnlyFile(tmpfile) == C_OK) {
            // 上报aof的子进程信息给主进程
            sendChildCOWInfo(CHILD_TYPE_AOF, "AOF rewrite");
            exitFromChild(0);
        } else {
            exitFromChild(1);
        }
    } else {
        /* Parent */
        if (childpid == -1) {
            closeChildInfoPipe();
            serverLog(LL_WARNING,
                      "Can't rewrite append only file in background: fork: %s",
                      strerror(errno));
            aofClosePipes();
            return C_ERR;
        }
        serverLog(LL_NOTICE,
                  "Background append only file rewriting started by pid %d", childpid);
        server.aof_rewrite_scheduled = 0;
        server.aof_rewrite_time_start = time(NULL);
        server.aof_child_pid = childpid;
        // 更新哈希重写策略（禁止重哈希）
        updateDictResizePolicy();
        /* We set appendseldb to -1 in order to force the next call to the
         * feedAppendOnlyFile() to issue a SELECT command, so the differences
         * accumulated by the parent into server.aof_rewrite_buf will start
         * with a SELECT statement and it will be safe to merge. */
        server.aof_selected_db = -1;
        replicationScriptCacheFlush();
        return C_OK;
    }
    return C_OK; /* unreached */
}
```



## 主进程

当进行aof重写时，主进程主要有几个操作：

- 更新重哈希的策略为尽量避免重哈希（避免页拷贝）
- 记录aof重写的开始时间
- 记录aof子进程的ID

```c
int rewriteAppendOnlyFileBackground(void) {
    ……
    // fork子进程
    if ((childpid = redisFork(CHILD_TYPE_AOF)) == 0) {
        ……
    } else {
        /* Parent */
        ……
        server.aof_rewrite_scheduled = 0;
      	// 记录aof重写的开始时间
        server.aof_rewrite_time_start = time(NULL);
      	// 记录aof子进程的ID
        server.aof_child_pid = childpid;
        // 更新哈希重写策略（主进程尽量避免重哈希）
        updateDictResizePolicy();
        server.aof_selected_db = -1;
        replicationScriptCacheFlush();
        return C_OK;
    }
    return C_OK; /* unreached */
}
```



## 子进程

子进程会将重哈希策略更新为禁止重哈希

```c
int redisFork(int purpose) {
    int childpid;
    long long start = ustime();
    // fork是copy-on-write
    if ((childpid = fork()) == 0) {
        /* Child */
        server.in_fork_child = purpose;
        setOOMScoreAdj(CONFIG_OOM_BGCHILD);
        // 设置信号处理线程
        setupChildSignalHandlers();
        // 设置重哈希策略（禁止重哈希）
        updateDictResizePolicy();
        // 关闭子进程不使用的资源（文件描述符等）
        closeClildUnusedResourceAfterFork();
    } else {
        ……
    }
    return childpid;
}
```



### 开始重写aof文件

重写aof文件的逻辑主要在`rewriteAppendOnlyFile`函数

```c
int rewriteAppendOnlyFileBackground(void) {
    ……
    if ((childpid = redisFork(CHILD_TYPE_AOF)) == 0) {
        char tmpfile[256];

        /* Child */
        // 设置进程名
        redisSetProcTitle("redis-aof-rewrite");
        // 设置cpu亲和性
        redisSetCpuAffinity(server.aof_rewrite_cpulist);
        // 初始化文件名
        snprintf(tmpfile, 256, "temp-rewriteaof-bg-%d.aof", (int) getpid());
      
      	// 重写aof文件
        if (rewriteAppendOnlyFile(tmpfile) == C_OK) {
            // 上报aof的子进程信息给主进程
            sendChildCOWInfo(CHILD_TYPE_AOF, "AOF rewrite");
            exitFromChild(0);
        } else {
            exitFromChild(1);
        }
    } else {
        ……
    }
    return C_OK; /* unreached */
}
```

- `rewriteAppendOnlyFile`函数

  1. 首先会创建一个临时文件`temp-rewriteaof-{pid}.aof`，将数据写入到临时文件中。

  2. 将当前数据库的数据写入到aof文件中。有两种格式：

  - 如果是混合模式，会把当前数据库的数据全部以rdb格式的方式写入
  - 否则，以aof格式的方式写入

  3. 再将redis数据库写入到aof文件的时候，如果写入的文件大小达到一定量`rdb->processed_bytes > processed+AOF_READ_DIFF_INTERVAL_BYTES`，会将主进程的部分命令先同步（通过管道）到子进程的缓冲区中。

  4. 当数据库中的数据（不包括aof期间的增量命令）已经全部同步完了，会立即刷盘。

  5. 之后，会将aof重写期间的增量命令，从主进程同步过来，放在子进程的缓冲区中`server.aof_child_diff`。

  6. 处理完成之后，告诉主进程，不需要再同步增量命令了，之后会等待主进程明确答复（已停止同步增量命令）。

  7. 跟主进程完成通信后，在通信期间也可能有增量命令的产生，所以会再一次从主进程同步增量命令到缓冲区。

  8. 接着，把缓冲区的数据全部一次性刷到aof文件中（aof格式）。将临时文件重命名为`temp-rewriteaof-bg-{pid}.aof`
  9. 最后，会把子进程的信息同步（通过管道）给主进程。然后退出（正常退出，退出码为0）

  ```c
  int rewriteAppendOnlyFile(char *filename) {
      rio aof;
      FILE *fp = NULL;
      char tmpfile[256];
      char byte;
  
      // 创建一个临时文件
      snprintf(tmpfile, 256, "temp-rewriteaof-%d.aof", (int) getpid());
      // 打开文件
      fp = fopen(tmpfile, "w");
      if (!fp) {
          ……
          return C_ERR;
      }
  
      server.aof_child_diff = sdsempty();
      // 初始化rio aof
      rioInitWithFile(&aof, fp);
  
      ……
  
      // 使用rdb快照写入到aof开头
      if (server.aof_use_rdb_preamble) {
          int error;
          if (rdbSaveRio(&aof, &error,RDBFLAGS_AOF_PREAMBLE,NULL) == C_ERR) {
              errno = error;
              goto werr;
          }
      } else {
          // 用aof的方式写入aof文件
          if (rewriteAppendOnlyFileRio(&aof) == C_ERR) goto werr;
      }
  
      // 存量已经同步完成了
      ……
      // 用户缓冲区的数据刷到操作系统
      if (fflush(fp) == EOF) goto werr;
      // 操作系统的数据立即刷到磁盘
      if (fsync(fileno(fp)) == -1) goto werr;
  
      ……
      int nodata = 0;
      mstime_t start = mstime();
      // 同步增量数据
      while (mstime() - start < 1000 && nodata < 20) {
          if (aeWait(server.aof_pipe_read_data_from_parent, AE_READABLE, 1) <= 0) {
              nodata++;
              continue;
          }
          nodata = 0;
          aofReadDiffFromParent();
      }
  
      ……
      // 发送ack告诉父进程不用增量同步了
      if (write(server.aof_pipe_write_ack_to_parent, "!", 1) != 1) goto werr;
      if (anetNonBlock(NULL, server.aof_pipe_read_ack_from_parent) != ANET_OK)
          goto werr;
      ……
      // 接收父进程不再新增增量命令的信息
      if (syncRead(server.aof_pipe_read_ack_from_parent, &byte, 1, 5000) != 1 ||
          byte != '!')
          goto werr;
      serverLog(LL_NOTICE, "Parent agreed to stop sending diffs. Finalizing AOF...");
  
      ……
      // 最后把所有的增量信息获取出来
      aofReadDiffFromParent();
  
      ……
      serverLog(LL_NOTICE,
                "Concatenating %.2f MB of AOF diff received from parent.",
                (double) sdslen(server.aof_child_diff) / (1024 * 1024));
      // 把缓存区的数据全部同步到aof文件中
      if (rioWrite(&aof, server.aof_child_diff, sdslen(server.aof_child_diff)) == 0)
          goto werr;
  
      ……
      // 刷到磁盘中
      if (fflush(fp)) goto werr;
      if (fsync(fileno(fp))) goto werr;
      if (fclose(fp)) {
          fp = NULL;
          goto werr;
      }
      fp = NULL;
  
      ……
      // 重命名文件
      if (rename(tmpfile, filename) == -1) {
          serverLog(LL_WARNING, "Error moving temp append only file on the final destination: %s", strerror(errno));
          unlink(tmpfile);
          stopSaving(0);
          return C_ERR;
      }
      serverLog(LL_NOTICE, "SYNC append only file rewrite performed");
      // 停止同步
      stopSaving(1);
      return C_OK;
  
  werr:
      serverLog(LL_WARNING, "Write error writing append only file on disk: %s", strerror(errno));
      if (fp) fclose(fp);
      unlink(tmpfile);
      stopSaving(0);
      return C_ERR;
  }
  ```

  至此，整个aof子进程的工作就完成了。



## 主进程增量命令同步

在aof文件重写期间，主进程还是对外提供服务，这期间会产生增量命令，需要同步给子进程

### 主进程

主进程处理客户端请求，执行客户端命令调用的函数是`call`函数

- 首先会调用`c->cmd->proc(c)`，执行具体的命令(set，get等)。
- 随后会传播这个命令，调用`propagate`函数

```c
void call(client *c, int flags) {
    ……
    // 调用函数，执行命令
    c->cmd->proc(c);
    ……
    // 传播这个命令
    if (flags & CMD_CALL_PROPAGATE &&
        (c->flags & CLIENT_PREVENT_PROP) != CLIENT_PREVENT_PROP) {
        ……
        // 传播命令
        if (propagate_flags != PROPAGATE_NONE && !(c->cmd->flags & CMD_MODULE))
            propagate(c->cmd, c->db->id, c->argv, c->argc, propagate_flags);
    }

    ……
}
```



- 传播命令`propagate`

  - 查看aof开关，开的话传播aof
  - 主从同步

  ```c
  void propagate(struct redisCommand *cmd, int dbid, robj **argv, int argc,
                 int flags) {
      // aof开关没关，并且标识传播aof
      if (server.aof_state != AOF_OFF && flags & PROPAGATE_AOF)
          feedAppendOnlyFile(cmd, dbid, argv, argc);
      if (flags & PROPAGATE_REPL)
          replicationFeedSlaves(server.slaves, dbid, argv, argc);
  }
  ```

- 传播aof

  - 构造aof命令格式
  - 写入到aof缓冲区
  - 写入到aof重写缓冲区（如果有aof子进程）

  ```c
  void feedAppendOnlyFile(struct redisCommand *cmd, int dictid, robj **argv, int argc) {
      sds buf = sdsempty();
      robj *tmpargv[3];
  
      // 不是当前aof选中的数据库
      if (dictid != server.aof_selected_db) {
          char seldb[64];
  
          // buf保存选中命令
          snprintf(seldb, sizeof(seldb), "%d", dictid);
          buf = sdscatprintf(buf, "*2\r\n$6\r\nSELECT\r\n$%lu\r\n%s\r\n",
                             (unsigned long) strlen(seldb), seldb);
          server.aof_selected_db = dictid;
      }
  
    	// 将要传播的命令写入到buf
      if (cmd->proc == expireCommand || cmd->proc == pexpireCommand ||
          cmd->proc == expireatCommand) {
          buf = catAppendOnlyExpireAtCommand(buf, cmd, argv[1], argv[2]);
      } else if (cmd->proc == setexCommand || cmd->proc == psetexCommand) {
          tmpargv[0] = createStringObject("SET", 3);
          tmpargv[1] = argv[1];
          tmpargv[2] = argv[3];
          buf = catAppendOnlyGenericCommand(buf, 3, tmpargv);
          decrRefCount(tmpargv[0]);
          buf = catAppendOnlyExpireAtCommand(buf, cmd, argv[1], argv[2]);
      } else if (cmd->proc == setCommand && argc > 3) {
          int i;
          robj *exarg = NULL, *pxarg = NULL;
          for (i = 3; i < argc; i++) {
              if (!strcasecmp(argv[i]->ptr, "ex")) exarg = argv[i + 1];
              if (!strcasecmp(argv[i]->ptr, "px")) pxarg = argv[i + 1];
          }
          serverAssert(!(exarg && pxarg));
  
          if (exarg || pxarg) {
              /* Translate SET [EX seconds][PX milliseconds] to SET and PEXPIREAT */
              buf = catAppendOnlyGenericCommand(buf, 3, argv);
              if (exarg)
                  buf = catAppendOnlyExpireAtCommand(buf, server.expireCommand, argv[1],
                                                     exarg);
              if (pxarg)
                  buf = catAppendOnlyExpireAtCommand(buf, server.pexpireCommand, argv[1],
                                                     pxarg);
          } else {
              // 查看最简单的set命令如何添加aof
              // set命令添加到buf中
              buf = catAppendOnlyGenericCommand(buf, argc, argv);
          }
      } else {
          buf = catAppendOnlyGenericCommand(buf, argc, argv);
      }
  
      // 放到主进程的aof缓冲区
      if (server.aof_state == AOF_ON)
          server.aof_buf = sdscatlen(server.aof_buf, buf, sdslen(buf));
  
      // 如果有aof子进程,添加到重写buffer中
      if (server.aof_child_pid != -1)
          aofRewriteBufferAppend((unsigned char *) buf, sdslen(buf));
  
      sdsfree(buf);
  }
  
  ```

- 写入aof重写缓冲区

  - 将aof格式的命令写入到aof重写块中`server.aof_rewrite_buf_blocks`
  - 添加epoll节点（注册完之后会在`aeProcessEvents`监听到可写之后调用`aofChildWriteDiffData`）

  ```c
  void aofRewriteBufferAppend(unsigned char *s, unsigned long len) {
      listNode *ln = listLast(server.aof_rewrite_buf_blocks);
      aofrwblock *block = ln ? ln->value : NULL;
  
      while (len) {
          // aof期间将命令保存到aof缓存区块
          if (block) {
              unsigned long thislen = (block->free < len) ? block->free : len;
              if (thislen) {
                  /* The current block is not already full. */
                  memcpy(block->buf + block->used, s, thislen);
                  block->used += thislen;
                  block->free -= thislen;
                  s += thislen;
                  len -= thislen;
              }
          }
  
          if (len) {
              /* First block to allocate, or need another block. */
              int numblocks;
  
              block = zmalloc(sizeof(*block));
              block->free = AOF_RW_BUF_BLOCK_SIZE;
              block->used = 0;
              // 将blocks块添加到链表尾
              listAddNodeTail(server.aof_rewrite_buf_blocks, block);
  
              /* Log every time we cross more 10 or 100 blocks, respectively
               * as a notice or warning. */
              // 链表块长度
              numblocks = listLength(server.aof_rewrite_buf_blocks);
              // 链表块等于10的整数倍
              if (((numblocks + 1) % 10) == 0) {
                  // 链表块过多记录日志
                  int level = ((numblocks + 1) % 100) == 0 ? LL_WARNING : LL_NOTICE;
                  serverLog(level, "Background AOF buffer size: %lu MB",
                            aofRewriteBufferSize() / (1024 * 1024));
              }
          }
      }
  
      /* Install a file event to send data to the rewrite child if there is
       * not one already. */
      if (!server.aof_stop_sending_diff &&
          aeGetFileEvents(server.el, server.aof_pipe_write_data_to_child) == 0) {
          // 添加到epoll节点中
          aeCreateFileEvent(server.el, server.aof_pipe_write_data_to_child,
                            AE_WRITABLE, aofChildWriteDiffData, NULL);
      }
  }
  ```

- 调用`aofChildWriteDiffData`函数

  - 从`server.aof_rewrite_buf_blocks`读取增量命令，通过管道`server.aof_pipe_write_data_to_child`发送到子进程

  ```c
  void aofChildWriteDiffData(aeEventLoop *el, int fd, void *privdata, int mask) {
      listNode *ln;
      aofrwblock *block;
      ssize_t nwritten;
      UNUSED(el);
      UNUSED(fd);
      UNUSED(privdata);
      UNUSED(mask);
  
      while (1) {
          // 读取blocks块
          ln = listFirst(server.aof_rewrite_buf_blocks);
          block = ln ? ln->value : NULL;
          // 如果aof停止了，或者block没数据了
          if (server.aof_stop_sending_diff || !block) {
              aeDeleteFileEvent(server.el, server.aof_pipe_write_data_to_child,
                                AE_WRITABLE);
              return;
          }
          if (block->used > 0) {
              // 将数据通过管道写入到子进程
              nwritten = write(server.aof_pipe_write_data_to_child,
                               block->buf, block->used);
              if (nwritten <= 0) return;
              // 更新block块
              memmove(block->buf, block->buf + nwritten, block->used - nwritten);
              block->used -= nwritten;
              block->free += nwritten;
          }
          // 如果没有空间了就删除这个节点
          if (block->used == 0) listDelNode(server.aof_rewrite_buf_blocks, ln);
      }
  }



### 子进程

子进程接收增量命令，放到缓冲区中

- 重写aof文件期间，当写入的文件到达一定量时，会调用`aofReadDiffFromParent`从`server.aof_pipe_read_data_from_parent`读取增量命令，保存到缓冲区中`server.aof_child_diff`

```c
ssize_t aofReadDiffFromParent(void) {
    char buf[65536]; /* Default pipe buffer size on most Linux systems. */
    ssize_t nread, total = 0;

    // 从管道读取aof期间的增量命令
    while ((nread =
            read(server.aof_pipe_read_data_from_parent, buf, sizeof(buf))) > 0) {
        // 将增量命令保存在aof_child_diff中
        server.aof_child_diff = sdscatlen(server.aof_child_diff, buf, nread);
        total += nread;
    }
    return total;
}
```



## 通知主进程停止同步增量命令

### 子进程

子进程在写完数据库的数据（不包括增量命令）到aof文件后，会告诉主进程停止同步增量命令

之后，阻塞等待主进程同步子进程（已停止同步增量命令），才开始处理后续的操作

```c
int rewriteAppendOnlyFile(char *filename) {
    ……

    // 使用rdb快照写入到aof开头
    if (server.aof_use_rdb_preamble) {
        int error;
        if (rdbSaveRio(&aof, &error,RDBFLAGS_AOF_PREAMBLE,NULL) == C_ERR) {
            errno = error;
            goto werr;
        }
    } else {
        // 用aof的方式写入aof文件
        if (rewriteAppendOnlyFileRio(&aof) == C_ERR) goto werr;
    }

    // 存量已经同步完成了
    // 用户缓冲区的数据刷到操作系统
    if (fflush(fp) == EOF) goto werr;
    // 操作系统的数据立即刷到磁盘
    if (fsync(fileno(fp)) == -1) goto werr;

    ……
      
    // 发送ack告诉父进程不用增量同步了
    if (write(server.aof_pipe_write_ack_to_parent, "!", 1) != 1) goto werr;
    if (anetNonBlock(NULL, server.aof_pipe_read_ack_from_parent) != ANET_OK)
        goto werr;
    
    // 接收父进程不再新增增量命令的信息
    if (syncRead(server.aof_pipe_read_ack_from_parent, &byte, 1, 5000) != 1 ||
        byte != '!')
        goto werr;
    serverLog(LL_NOTICE, "Parent agreed to stop sending diffs. Finalizing AOF...");

    ……
}
```



### 主进程

主进程在创建管道（三对）的时候，把停止同步增量命令的读管道添加到epoll中。处理函数为`aofChildPipeReadable`

```c
int aofCreatePipes(void) {
    // 创建三对管道
    int fds[6] = {-1, -1, -1, -1, -1, -1};
    int j;

    if (pipe(fds) == -1) goto error; /* parent -> children data. */
    if (pipe(fds + 2) == -1) goto error; /* children -> parent ack. */
    if (pipe(fds + 4) == -1) goto error; /* parent -> children ack. */
    /* Parent -> children data is non blocking. */
    // 将第一对管道设置为非阻塞
    if (anetNonBlock(NULL, fds[0]) != ANET_OK) goto error;
    if (anetNonBlock(NULL, fds[1]) != ANET_OK) goto error;
  
    // 停止同步增量命令管道，将fds[2]加入到epoll中
    if (aeCreateFileEvent(server.el, fds[2], AE_READABLE, aofChildPipeReadable, NULL) == AE_ERR) goto error;

    // 设置管道
    server.aof_pipe_write_data_to_child = fds[1];
    server.aof_pipe_read_data_from_parent = fds[0];
    server.aof_pipe_write_ack_to_parent = fds[3];
  	// 停止同步增量命令管道
    server.aof_pipe_read_ack_from_child = fds[2];
    server.aof_pipe_write_ack_to_child = fds[5];
    server.aof_pipe_read_ack_from_parent = fds[4];
    server.aof_stop_sending_diff = 0;
    return C_OK;

error:
    serverLog(LL_WARNING, "Error opening /setting AOF rewrite IPC pipes: %s",
              strerror(errno));
    for (j = 0; j < 6; j++) if (fds[j] != -1) close(fds[j]);
    return C_ERR;
}
```

处理停止同步增量命令函数`aofChildPipeReadable`

- 将`server.aof_stop_sending_diff`设置为1
- 这样，主进程在`传播命令时`就不会把`server.aof_rewrite_buf_blocks`中的数据同步给子进程了（在此期间，增量命令依然会写入server.aof_rewrite_buf_blocks）
- 设置完成之后，告诉子进程`write(server.aof_pipe_write_ack_to_child, "!", 1)`已停止同步增量数据

```c
void aofChildPipeReadable(aeEventLoop *el, int fd, void *privdata, int mask) {
    char byte;
    UNUSED(el);
    UNUSED(privdata);
    UNUSED(mask);

    if (read(fd, &byte, 1) == 1 && byte == '!') {
        serverLog(LL_NOTICE, "AOF rewrite child asks to stop sending diffs.");
        server.aof_stop_sending_diff = 1;
        if (write(server.aof_pipe_write_ack_to_child, "!", 1) != 1) {
            /* If we can't send the ack, inform the user, but don't try again
             * since in the other side the children will use a timeout if the
             * kernel can't buffer our write, or, the children was
             * terminated. */
            serverLog(LL_WARNING, "Can't send ACK to AOF child: %s",
                      strerror(errno));
        }
    }
    /* Remove the handler since this can be called only one time during a
     * rewrite. */
    aeDeleteFileEvent(server.el, server.aof_pipe_read_ack_from_child,AE_READABLE);
}
```

主进程告诉子进程停止同步增量数据之后，主进程依然还会有增量数据。这个时候的增量数据由主进程来处理



## 监听子进程退出

当aof重写子进程写完aof文件后，文件还没替换。这个时候，主进程还有增量命令，所以在停止同步增量到子进程，到aof文件替换期间，aof重写文件还缺了这部分的增量命令（增量命令都保存在`server.aof_rewrite_buf_blocks`中）。主进程会监听子进程退出，随后处理这部分增量命令

```c
int serverCron(struct aeEventLoop *eventLoop, long long id, void *clientData) {
    ……
    // 检查子进程是否已完成
    if (hasActiveChildProcess() || ldbPendingChildren()) {
        checkChildrenDone();
    } else {
        ……
    }
  
    ……
      
    return 1000 / server.hz;
}
```

调用`checkChildrenDone`函数

- 调用`backgroundRewriteDoneHandler`处理aof子进程退出

```c
void checkChildrenDone(void) {
    int statloc;
    pid_t pid;

    // 检查是否有子进程退出，获取子进程ID
    if ((pid = wait3(&statloc,WNOHANG,NULL)) != 0) {
        // 获取退出码
        int exitcode = WEXITSTATUS(statloc);
        int bysignal = 0;

        // 子进程是否是被杀死的
        if (WIFSIGNALED(statloc)) bysignal = WTERMSIG(statloc);     // 取出杀死它的信号编码

        /* sigKillChildHandler catches the signal and calls exit(), but we
         * must make sure not to flag lastbgsave_status, etc incorrectly.
         * We could directly terminate the child process via SIGUSR1
         * without handling it, but in this case Valgrind will log an
         * annoying error. */
        // 主进程杀死的
        if (exitcode == SERVER_CHILD_NOERROR_RETVAL) {
            bysignal = SIGUSR1;     // 标记成"被我自己杀的"
            exitcode = 1;           // 伪装成普通的失败退出（防止误报成功）
        }

        if (pid == -1) {
            ……
        } else if (pid == server.rdb_child_pid) {
            ……
        } else if (pid == server.aof_child_pid) {       // 处理aof子进程退出
            backgroundRewriteDoneHandler(exitcode, bysignal);
            if (!bysignal && exitcode == 0) receiveChildInfo();
        } else if (pid == server.module_child_pid) {
            ……
        } else {
            ……
        }
      
      	// 更新重哈希策略
        updateDictResizePolicy();
        closeChildInfoPipe();
    }
}
```

调用`backgroundRewriteDoneHandler`函数

- 拿到aof子进程重写的文件
- 把增量命令写入到重写文件中
- 覆盖旧的aof文件
- 根据aof策略采取不同的动作
- 将aof_buf清空

```c
void backgroundRewriteDoneHandler(int exitcode, int bysignal) {
    // 正常退出
    if (!bysignal && exitcode == 0) {
        int newfd, oldfd;
        char tmpfile[256];
        ……
        // 打开aof子进程写入的aof文件
        snprintf(tmpfile, 256, "temp-rewriteaof-bg-%d.aof",
                 (int) server.aof_child_pid);
        newfd = open(tmpfile,O_WRONLY | O_APPEND);
        if (newfd == -1) {
            serverLog(LL_WARNING,
                      "Unable to open the temporary AOF produced by the child: %s", strerror(errno));
            goto cleanup;
        }

        // 将增量命令写入到子进程的aof文件
        if (aofRewriteBufferWrite(newfd) == -1) {
            serverLog(LL_WARNING,
                      "Error trying to flush the parent diff to the rewritten AOF: %s", strerror(errno));
            close(newfd);
            goto cleanup;
        }
        ……
          
        // 重命名覆盖aof文件
        if (rename(tmpfile, server.aof_filename) == -1) {
            serverLog(LL_WARNING,
                      "Error trying to rename the temporary AOF file %s into %s: %s",
                      tmpfile,
                      server.aof_filename,
                      strerror(errno));
            close(newfd);
            if (oldfd != -1) close(oldfd);
            goto cleanup;
        }
        ……

        if (server.aof_fd == -1) {
            /* AOF disabled, we don't need to set the AOF file descriptor
             * to this new file, so we can close it. */
            close(newfd);
        } else {
            /* AOF enabled, replace the old fd with the new one. */
            oldfd = server.aof_fd;
            server.aof_fd = newfd;
            // 刷盘
            if (server.aof_fsync == AOF_FSYNC_ALWAYS)
                redis_fsync(newfd);
            else if (server.aof_fsync == AOF_FSYNC_EVERYSEC)        // 每秒刷盘，更新任务
                aof_background_fsync(newfd);
            server.aof_selected_db = -1;
            aofUpdateCurrentSize();
            // 更新重写aof文件的大小
            server.aof_rewrite_base_size = server.aof_current_size;
            // aof同步文件的偏移量
            server.aof_fsync_offset = server.aof_current_size;

            /* Clear regular AOF buffer since its contents was just written to
             * the new AOF from the background rewrite buffer. */
            // 将aof_buf的数据清空，因为server.aof_rewrite_buf_blocks已经有这部分数据了，已经写入到新文件中了，不然就会有重复命令了
            sdsfree(server.aof_buf);
            server.aof_buf = sdsempty();
        }

        server.aof_lastbgrewrite_status = C_OK;

        serverLog(LL_NOTICE, "Background AOF rewrite finished successfully");
        /* Change state from WAIT_REWRITE to ON if needed */
        if (server.aof_state == AOF_WAIT_REWRITE)
            server.aof_state = AOF_ON;

        /* Asynchronously close the overwritten AOF. */
        if (oldfd != -1) bioCreateBackgroundJob(BIO_CLOSE_FILE, (void *) (long) oldfd,NULL,NULL);       // 异步关闭旧文件

        serverLog(LL_VERBOSE,
                  "Background AOF rewrite signal handler took %lldus", ustime() - now);
    } else if (!bysignal && exitcode != 0) {
        server.aof_lastbgrewrite_status = C_ERR;

        serverLog(LL_WARNING,
                  "Background AOF rewrite terminated with error");
    } else {
        /* SIGUSR1 is whitelisted, so we have a way to kill a child without
         * triggering an error condition. */
        if (bysignal != SIGUSR1)
            server.aof_lastbgrewrite_status = C_ERR;

        serverLog(LL_WARNING,
                  "Background AOF rewrite terminated by signal %d", bysignal);
    }

cleanup:
    aofClosePipes();
    aofRewriteBufferReset();
    aofRemoveTempFile(server.aof_child_pid);
    server.aof_child_pid = -1;
    server.aof_rewrite_time_last = time(NULL) - server.aof_rewrite_time_start;
    server.aof_rewrite_time_start = -1;
    /* Schedule a new rewrite if we are waiting for it to switch the AOF ON. */
    if (server.aof_state == AOF_WAIT_REWRITE)
        server.aof_rewrite_scheduled = 1;
}
```

到这里，整个aof重写就完成了





# AOF文件写入

这是正常的aof文件记录的情况。

在执行完客户端命令后，会将请求放到aof缓存中。之后在将响应返回客户端之前，会把aof缓存刷到磁盘中（根据刷盘策略），然后才将响应写回客户端。

- 执行完客户端命令，记录aof缓存

  ```c
  void call(client *c, int flags) {
      ……
        
      // 调用函数，执行客户端命令
      c->cmd->proc(c);
    
      ……
  
      // 传播这个命令
      if (flags & CMD_CALL_PROPAGATE &&
          (c->flags & CLIENT_PREVENT_PROP) != CLIENT_PREVENT_PROP) {
          // 传播命令
          if (propagate_flags != PROPAGATE_NONE && !(c->cmd->flags & CMD_MODULE))
              propagate(c->cmd, c->db->id, c->argv, c->argc, propagate_flags);
      }
  
      ……
  }
  
  void propagate(struct redisCommand *cmd, int dbid, robj **argv, int argc,
                 int flags) {
      // aof开关没关，并且标识传播aof
      if (server.aof_state != AOF_OFF && flags & PROPAGATE_AOF)
          feedAppendOnlyFile(cmd, dbid, argv, argc);
      ……
  }
  
  void feedAppendOnlyFile(struct redisCommand *cmd, int dictid, robj **argv, int argc) {
      sds buf = sdsempty();
      robj *tmpargv[3];
  
      ……
        
      // 不是当前aof选中的数据库
      if (dictid != server.aof_selected_db) {
          char seldb[64];
  
          // buf保存选中命令
          snprintf(seldb, sizeof(seldb), "%d", dictid);
          buf = sdscatprintf(buf, "*2\r\n$6\r\nSELECT\r\n$%lu\r\n%s\r\n",
                             (unsigned long) strlen(seldb), seldb);
          server.aof_selected_db = dictid;
      }
  
    	// 命令保存在buf中
      if (cmd->proc == expireCommand || cmd->proc == pexpireCommand ||
          cmd->proc == expireatCommand) {
          buf = catAppendOnlyExpireAtCommand(buf, cmd, argv[1], argv[2]);
      } else if (cmd->proc == setexCommand || cmd->proc == psetexCommand) {
          tmpargv[0] = createStringObject("SET", 3);
          tmpargv[1] = argv[1];
          tmpargv[2] = argv[3];
          buf = catAppendOnlyGenericCommand(buf, 3, tmpargv);
          decrRefCount(tmpargv[0]);
          buf = catAppendOnlyExpireAtCommand(buf, cmd, argv[1], argv[2]);
      } else if (cmd->proc == setCommand && argc > 3) {
          int i;
          robj *exarg = NULL, *pxarg = NULL;
          for (i = 3; i < argc; i++) {
              if (!strcasecmp(argv[i]->ptr, "ex")) exarg = argv[i + 1];
              if (!strcasecmp(argv[i]->ptr, "px")) pxarg = argv[i + 1];
          }
          serverAssert(!(exarg && pxarg));
  
          if (exarg || pxarg) {
              buf = catAppendOnlyGenericCommand(buf, 3, argv);
              if (exarg)
                  buf = catAppendOnlyExpireAtCommand(buf, server.expireCommand, argv[1],
                                                     exarg);
              if (pxarg)
                  buf = catAppendOnlyExpireAtCommand(buf, server.pexpireCommand, argv[1],
                                                     pxarg);
          } else {
              // 查看最简单的set命令如何添加aof
              // set命令添加到buf中
              buf = catAppendOnlyGenericCommand(buf, argc, argv);
          }
      } else {
          buf = catAppendOnlyGenericCommand(buf, argc, argv);
      }
  
    
    
      // 命令写到aof缓存中
      if (server.aof_state == AOF_ON)
          server.aof_buf = sdscatlen(server.aof_buf, buf, sdslen(buf));
  
      ……
  
      sdsfree(buf);
  }
  ```

- 在写回客户端响应之前，刷盘（根据策略）

  ```c
  void beforeSleep(struct aeEventLoop *eventLoop) {
      ……
  
      // 先刷盘，再响应客户端
      flushAppendOnlyFile(0);
  
      // 将响应值写回客户端
      handleClientsWithPendingWritesUsingThreads();
  
      ……
  }
  
  ```

  刷盘

  - 刷盘策略为`ALWAYS`，直接刷盘
  - 刷盘策略为`EVERYSEC`，满足条件后交给后台线程刷盘

  ```c
  void flushAppendOnlyFile(int force) {
      ……
  
  try_fsync:
      /* Don't fsync if no-appendfsync-on-rewrite is set to yes and there are
       * children doing I/O in the background. */
      // aof期间不刷盘（配置文件配置） && 有子进程就不刷盘
      if (server.aof_no_fsync_on_rewrite && hasActiveChildProcess())
          return;
  
      // 根据不同的刷盘策略去处理
      if (server.aof_fsync == AOF_FSYNC_ALWAYS) {
          latencyStartMonitor(latency);
          redis_fsync(server.aof_fd); /* Let's try to get this data on the disk */
          latencyEndMonitor(latency);
          latencyAddSampleIfNeeded("aof-fsync-always", latency);
          server.aof_fsync_offset = server.aof_current_size;
          server.aof_last_fsync = server.unixtime;
      } else if ((server.aof_fsync == AOF_FSYNC_EVERYSEC &&       // 每秒刷盘
                  server.unixtime > server.aof_last_fsync)) {     // 超过1秒
          if (!sync_in_progress) {            // 没有后台线程就创建
              aof_background_fsync(server.aof_fd);
              server.aof_fsync_offset = server.aof_current_size;
          }
          server.aof_last_fsync = server.unixtime;
      }
  }
  ```

  











