# Info

版本信息：Redis 6.0



# RDB

在一定时间内，达到一定的修改次数。就会调用`rdbSaveBackground`函数触发rdb持久化保存

```c
int serverCron(struct aeEventLoop *eventLoop, long long id, void *clientData) {
    ……

    // 检查子进程是否已完成
    if (hasActiveChildProcess() || ldbPendingChildren()) {
        checkChildrenDone();
    } else {
      	// RDB持久化
        for (j = 0; j < server.saveparamslen; j++) {
            struct saveparam *sp = server.saveparams + j;

            if (server.dirty >= sp->changes &&                  // 修改次数达到条件
                server.unixtime - server.lastsave > sp->seconds &&      // 在一定时间内
                (server.unixtime - server.lastbgsave_try >
                 CONFIG_BGSAVE_RETRY_DELAY ||
                 server.lastbgsave_status == C_OK)) {
                serverLog(LL_NOTICE, "%d changes in %d seconds. Saving...",
                          sp->changes, (int) sp->seconds);
                rdbSaveInfo rsi, *rsiptr;
                rsiptr = rdbPopulateSaveInfo(&rsi);
              	// 持久化
                rdbSaveBackground(server.rdb_filename, rsiptr);
                break;
            }
        }
    }
  
  	……
}
```



# RDB文件生成

RDB文件生成的流程与AOF文件重写的流程差不多，但是比AOF简单得多，只是生成一个快照，不涉及生成期间增量命令的处理。

- fork一个子进程来执行RDB持久化
- 主进程更新重哈希策略（rdb生成期间尽量避免重哈希【避免页拷贝】）

```c
int rdbSaveBackground(char *filename, rdbSaveInfo *rsi) {
    pid_t childpid;

    // 有子进程就不执行
    if (hasActiveChildProcess()) return C_ERR;

    server.dirty_before_bgsave = server.dirty;
    server.lastbgsave_try = time(NULL);
    openChildInfoPipe();

    if ((childpid = redisFork(CHILD_TYPE_RDB)) == 0) {
        int retval;

        /* Child */
        redisSetProcTitle("redis-rdb-bgsave");
        redisSetCpuAffinity(server.bgsave_cpulist);
        // 保存rdb文件
        retval = rdbSave(filename,rsi);
        if (retval == C_OK) {
            sendChildCOWInfo(CHILD_TYPE_RDB, "RDB");
        }
        exitFromChild((retval == C_OK) ? 0 : 1);
    } else {
        /* Parent */
        if (childpid == -1) {
            closeChildInfoPipe();
            server.lastbgsave_status = C_ERR;
            serverLog(LL_WARNING,"Can't save in background: fork: %s",
                strerror(errno));
            return C_ERR;
        }
        serverLog(LL_NOTICE,"Background saving started by pid %d",childpid);
        server.rdb_save_time_start = time(NULL);
        server.rdb_child_pid = childpid;
        server.rdb_child_type = RDB_CHILD_TYPE_DISK;
        // 更新重哈希策略（设置为禁止重哈希）
        updateDictResizePolicy();
        return C_OK;
    }
    return C_OK; /* unreached */
}
```



## 保存RDB文件

子进程保存RDB文件

- 打开临时文件
- 将redis内存数据库以rdb格式写入到临时文件
- 刷盘
- 替换rdb文件

```c
int rdbSave(char *filename, rdbSaveInfo *rsi) {
    ……

    // 打开临时文件
    snprintf(tmpfile,256,"temp-%d.rdb", (int) getpid());
    fp = fopen(tmpfile,"w");
    if (!fp) {
        char *cwdp = getcwd(cwd,MAXPATHLEN);
        serverLog(LL_WARNING,
            "Failed opening the RDB file %s (in server root dir %s) "
            "for saving: %s",
            filename,
            cwdp ? cwdp : "unknown",
            strerror(errno));
        return C_ERR;
    }

    ……

    // 将内存数据库已rdb格式写入到rdb文件
    if (rdbSaveRio(&rdb,&error,RDBFLAGS_NONE,rsi) == C_ERR) {
        errno = error;
        goto werr;
    }

    // 刷盘
    if (fflush(fp)) goto werr;
    if (fsync(fileno(fp))) goto werr;
    if (fclose(fp)) { fp = NULL; goto werr; }
    fp = NULL;
    
    // 重命名文件
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
        stopSaving(0);
        return C_ERR;
    }

    serverLog(LL_NOTICE,"DB saved on disk");
    server.dirty = 0;
    server.lastsave = time(NULL);
    server.lastbgsave_status = C_OK;
    stopSaving(1);
    return C_OK;

werr:
    serverLog(LL_WARNING,"Write error saving DB on disk: %s", strerror(errno));
    if (fp) fclose(fp);
    unlink(tmpfile);
    stopSaving(0);
    return C_ERR;
}
```



## 监控子进程退出

同AOF一样，在子进程退出之后，redis会调用`backgroundSaveDoneHandler`函数进行处理。

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

        // 主进程杀死的
        if (exitcode == SERVER_CHILD_NOERROR_RETVAL) {
            bysignal = SIGUSR1;     // 标记成"被我自己杀的"
            exitcode = 1;           // 伪装成普通的失败退出（防止误报成功）
        }

        if (pid == -1) {
            ……
        } else if (pid == server.rdb_child_pid) {       // 处理rdb子进程退出
            backgroundSaveDoneHandler(exitcode, bysignal);
            if (!bysignal && exitcode == 0) receiveChildInfo();
        } else if (pid == server.aof_child_pid) {       // 处理aof子进程退出
            ……
        } else if (pid == server.module_child_pid) {
            ……
        } else {
            ……
        }
        ……
    }
}
```

- `backgroundSaveDoneHandler`

  - 更新dirty数

  ```c
  void backgroundSaveDoneHandler(int exitcode, int bysignal) {
      int type = server.rdb_child_type;
      switch(server.rdb_child_type) {
      case RDB_CHILD_TYPE_DISK:        // 本地持久化
          backgroundSaveDoneHandlerDisk(exitcode,bysignal);
          break;
      case RDB_CHILD_TYPE_SOCKET:     // 主从同步
          ……
      default:
          ……
      }
  
      server.rdb_child_pid = -1;
      server.rdb_child_type = RDB_CHILD_TYPE_NONE;
      server.rdb_save_time_last = time(NULL)-server.rdb_save_time_start;
      server.rdb_save_time_start = -1;
      ……
  }
  
  // 更新dirty数
  static void backgroundSaveDoneHandlerDisk(int exitcode, int bysignal) {
      if (!bysignal && exitcode == 0) {
          serverLog(LL_NOTICE,
              "Background saving terminated with success");
          server.dirty = server.dirty - server.dirty_before_bgsave;
          server.lastsave = time(NULL);
          server.lastbgsave_status = C_OK;
      } else if (!bysignal && exitcode != 0) {
          ……
      } else {
          ……
      }
  }
  ```

  