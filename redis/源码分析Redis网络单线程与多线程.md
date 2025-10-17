# 源码分析Redis网络单线程与多线程

# Redis info

版本：6.0

系统：linux

默认单线程模型，开启多线程要在配置文件配置

# 单线程

## 相关主流程

涉及到redis网络处理的主要流程有两步：

- 初始化server
- 循环事件

```c
int main(int argc, char **argv) {
  
  	……
  
    //初始化server结构体，包括文件描述符的创建
    initServer();

  	……

    // 循环事件
    aeMain(server.el);
    
  	……
    
    return 0;
}
```



## 初始化server

server在redis中是一个全局对象

- server数据结构

  ```c
  struct redisServer {
    // 事件循环器
    aeEventLoop *el;
    ……
    // tcp监听的端口
    int port;
    ……
    // 保存文件描述符
    int ipfd[CONFIG_BINDADDR_MAX]; /* TCP socket file descriptors */
    // 从0开始，下一个ipfd要保存的下标
    int ipfd_count;
    ……
    // 待响应到客户端的数据
    list *clients_pending_write;
  }
  ```



### server的初始化

  server初始化会先初始化事件循环器，然后进行服务端tcp编程操作（create_socket、bind、listen），同时将对应的socket绑定到epoll中，设置socket对应的事件处理函数

  - 创建事件循环器
  - 监听sokcet（tcp编程）：这时候可以接收客户端的连接了
    - 生成的socket文件描述符会保存在server.ipfd中
  - 将文件描述符封装成aeFileEvent对象，并将文件描述符添加到epoll中。
    - 同时设置读事件触发后，对应的读事件处理函数
  - 设置事件循环器的beforeSleep和afterSleep函数

  ```c
  void initServer(void) {
   
    	……	
    
      //创建事件循环器，创建epoll对象
      server.el = aeCreateEventLoop(server.maxclients + CONFIG_FDSET_INCR);
      if (server.el == NULL) {
          serverLog(LL_WARNING,
                    "Failed creating the event loop. Error message: '%s'",
                    strerror(errno));
          exit(1);
      }
    
    	……
  
      // 监听socket
      if (server.port != 0 &&
          listenToPort(server.port, server.ipfd, &server.ipfd_count) == C_ERR)
          exit(1);
  		……
  
      // 添加socket到epoll
      for (j = 0; j < server.ipfd_count; j++) {
          if (aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE,
                                acceptTcpHandler,NULL) == AE_ERR) {
              serverPanic(
                  "Unrecoverable error creating server.ipfd file event.");
          }
      }
      
      // 设置beforeSleep函数，事件处理前调用
      aeSetBeforeSleepProc(server.el, beforeSleep);
      // 设置afterSleep函数，事件处理后调用
      aeSetAfterSleepProc(server.el, afterSleep);
  
    	……
  }
  ```

  

#### 创建事件循环器

主要是创建epoll对象，其他的都是默认值

- 结构

  ```c
  typedef struct aeEventLoop {
      // 初始化的时候是-1
      // 保存的是当前redis中最大的文件描述符
      int maxfd;
      // 文件描述符的最大数量
      int setsize;
      ……
      // 注册事件，epoll节点注册的事件（文件描述符）
      aeFileEvent *events;
      // 激活事件，epoll有事件处理，读取出来之后会保存到这里
      aeFiredEvent *fired; /* Fired events */
      ……
      //epoll对象，epoll的根在这里
      void *apidata;
      // beforeSleep
      aeBeforeSleepProc *beforesleep;
      // afterSleep
      aeBeforeSleepProc *aftersleep;
      int flags;
  } aeEventLoop;
  ```
  
- 事件循环器创建

  ```c
  aeEventLoop *aeCreateEventLoop(int setsize) {
      aeEventLoop *eventLoop;
    	// aeEventLoop属性的默认值设置
      ……
      // 创建epoll对象，设置到eventLoop->apidata
      if (aeApiCreate(eventLoop) == -1) goto err;
    
     	// 默认空对象
      for (i = 0; i < setsize; i++)
          eventLoop->events[i].mask = AE_NONE; //设置事件状态
      return eventLoop;
  
  err:
      if (eventLoop) {
          zfree(eventLoop->events);
          zfree(eventLoop->fired);
          zfree(eventLoop);
      }
      return NULL;
  }
  ```

  

#### 监听socket

- 创建监听socket
- 执行bind和listen操作
- 监听socket的文件描述符保存到server.fds

```c
int listenToPort(int port, int *fds, int *count) {
    int j;

    // bindaddr_count = 0，for循环只会执行一次
    if (server.bindaddr_count == 0) server.bindaddr[0] = NULL;
    for (j = 0; j < server.bindaddr_count || j == 0; j++) {
        // 建立连接监听
        if (server.bindaddr[j] == NULL) {
            int unsupported = 0;
            // 监听，并把文件描述符返回(创建socket,bind,listen)，这个是ipv6
            fds[*count] = anetTcp6Server(server.neterr, port,NULL,
                                         server.tcp_backlog);
            if (fds[*count] != ANET_ERR) {
                //设置为非阻塞
                anetNonBlock(NULL, fds[*count]);
                // 修改下一个要保存的文件描述符位置
                (*count)++;
            } else if (errno == EAFNOSUPPORT) {
                ……
            }

            if (*count == 1 || unsupported) {
                //支持tcp6也支持tcp4，这个是ipv4
                fds[*count] = anetTcpServer(server.neterr, port,NULL,
                                            server.tcp_backlog);
                if (fds[*count] != ANET_ERR) {
                    anetNonBlock(NULL, fds[*count]);
                    (*count)++;
                } else if (errno == EAFNOSUPPORT) {
                    ……
                }
            }
            // 从这里直接退出
            if (*count + unsupported == 2) break;
        }
      
      	……
    }
    return C_OK;
}
```



#### 添加监听socket到epoll

监听socket的时候创建了两个socket（ipv4和ipv6），保存在server.fds，server.ipfd_count对应的值也会增加

```c
void initServer(void) {
		……
    // 添加socket到epoll
    for (j = 0; j < server.ipfd_count; j++) {
        if (aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE,
                              acceptTcpHandler,NULL) == AE_ERR) {
            serverPanic(
                "Unrecoverable error creating server.ipfd file event.");
        }
    }
  	……
}
```

- 添加socket到epoll

  - 将socket添加epoll中，监听socket的读事件
  - 在事件循环器中拿到文件描述符对应的文件事件对象
  - 设置文件事件对象的属性（可读的，读处理函数：acceptTcpHandler）

  ```c
  int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask,
                        aeFileProc *proc, void *clientData) {
      if (fd >= eventLoop->setsize) {
          errno = ERANGE;
          return AE_ERR;
      }
      aeFileEvent *fe = &eventLoop->events[fd];
  
      // 将文件描述符添加到epoll中
      if (aeApiAddEvent(eventLoop, fd, mask) == -1)
          return AE_ERR;
      fe->mask |= mask;
      // 保存读事件处理函数
      if (mask & AE_READABLE) fe->rfileProc = proc;
      // 保存写事件处理函数
      if (mask & AE_WRITABLE) fe->wfileProc = proc;
      fe->clientData = clientData;
      // 更新最大文件描述符
      if (fd > eventLoop->maxfd)
          eventLoop->maxfd = fd;
      return AE_OK;
  }
  ```





## 事件循环处理

不断的循环调用aeProcessEvents

```c
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        aeProcessEvents(eventLoop, AE_ALL_EVENTS |
                                   AE_CALL_BEFORE_SLEEP |
                                   AE_CALL_AFTER_SLEEP);
    }
}
```



### 事件处理

从epoll上取出待处理的事件，遍历这些事件，一个个处理

- 调用beforesleep函数
- 处理epoll上的事件
- 调用aftersleep函数

```c
int aeProcessEvents(aeEventLoop *eventLoop, int flags) {
    int processed = 0, numevents;

    // 命中第一个条件，直接进入
    if (eventLoop->maxfd != -1 ||
        ((flags & AE_TIME_EVENTS) && !(flags & AE_DONT_WAIT))) {
        int j;
        
      	……

        if (eventLoop->beforesleep != NULL && flags & AE_CALL_BEFORE_SLEEP)
            eventLoop->beforesleep(eventLoop);

        // 返回事件的数量
        numevents = aeApiPoll(eventLoop, tvp);

        /* 回调事件 After sleep callback. */
        if (eventLoop->aftersleep != NULL && flags & AE_CALL_AFTER_SLEEP)
            eventLoop->aftersleep(eventLoop);

        for (j = 0; j < numevents; j++) {
            //拿到事件
            aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];
            int mask = eventLoop->fired[j].mask;
            int fd = eventLoop->fired[j].fd;
            int fired = 0;

          	//调用反转，忽略，这里不会有这种情况
            int invert = fe->mask & AE_BARRIER;

            if (!invert && fe->mask & mask & AE_READABLE) {
                //有读事件,调用读处理函数
                fe->rfileProc(eventLoop, fd, fe->clientData, mask);
                fired++;
                // 刷新一下，events的尺寸可能调整
                fe = &eventLoop->events[fd];
            }

            ……

            processed++;
        }
    }
    ……

    return processed;
}
```



### 处理epoll上的事件（针对监听socket）

#### 获取epoll上的事件

- 调用epoll_wait获取事件
- 将对应的文件描述符封装成aeFiredEvent对象，保存到eventLoop->fired中

```c
static int aeApiPoll(aeEventLoop *eventLoop, struct timeval *tvp) {
    aeApiState *state = eventLoop->apidata;
    int retval, numevents = 0;

    // 获取事件
    retval = epoll_wait(state->epfd,state->events,eventLoop->setsize,
            tvp ? (tvp->tv_sec*1000 + tvp->tv_usec/1000) : -1);
    // 事件个数大于0
    if (retval > 0) {
        int j;

        numevents = retval;
        for (j = 0; j < numevents; j++) {
            int mask = 0;
            struct epoll_event *e = state->events+j;

            // 有数据可读取
            if (e->events & EPOLLIN) mask |= AE_READABLE;
            // 有数据可写
            if (e->events & EPOLLOUT) mask |= AE_WRITABLE;
            if (e->events & EPOLLERR) mask |= AE_WRITABLE|AE_READABLE;
            if (e->events & EPOLLHUP) mask |= AE_WRITABLE|AE_READABLE;
            eventLoop->fired[j].fd = e->data.fd;
            eventLoop->fired[j].mask = mask;
        }
    }
    return numevents;
}
```



#### 处理epoll上的事件

- 调用文件事件注册的读处理函数

```c
int aeProcessEvents(aeEventLoop *eventLoop, int flags) {
    int processed = 0, numevents;

    // 命中第一个条件，直接进入
    if (eventLoop->maxfd != -1 ||
        ((flags & AE_TIME_EVENTS) && !(flags & AE_DONT_WAIT))) {
        int j;
        
      	……

        // 返回事件的数量
        numevents = aeApiPoll(eventLoop, tvp);

        ……

        for (j = 0; j < numevents; j++) {
            //拿到事件
            aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];
            int mask = eventLoop->fired[j].mask;
            int fd = eventLoop->fired[j].fd;
            int fired = 0;

          	//调用反转，忽略，这里不会有这种情况
            int invert = fe->mask & AE_BARRIER;

            if (!invert && fe->mask & mask & AE_READABLE) {
                //有读事件,调用读处理函数
                fe->rfileProc(eventLoop, fd, fe->clientData, mask);
                fired++;
                // 刷新一下，events的尺寸可能调整
                fe = &eventLoop->events[fd];
            }

            ……

            processed++;
        }
    }
    ……

    return processed;
}
```



- 对于监听socket来说，读处理函数是acceptTcpHandler，这个是在将socket添加到epoll的时候注册的函数

  - 监听socket触发，说明有连接进来了
  - 调用accept函数，获取 已连接socket
  - 将已连接socket注册到epoll中，并设置读处理函数

  ```c
  void acceptTcpHandler(aeEventLoop *el, int fd, void *privdata, int mask) {
      int cport, cfd, max = MAX_ACCEPTS_PER_CALL;
      char cip[NET_IP_STR_LEN];
      UNUSED(el);
      UNUSED(mask);
      UNUSED(privdata);
  
      // 最多accept的次数
      while (max--) {
          //anetTcpAccept调用accept函数拿到文件描述符
          cfd = anetTcpAccept(server.neterr, fd, cip, sizeof(cip), &cport);
          if (cfd == ANET_ERR) {
              if (errno != EWOULDBLOCK)
                  serverLog(LL_WARNING,
                            "Accepting client connection: %s", server.neterr);
              return;
          }
          serverLog(LL_VERBOSE, "Accepted %s:%d", cip, cport);
          // 创建一个conn socket
          // 将文件描述符添加到epoll，并设置相关的处理函数
          acceptCommonHandler(connCreateAcceptedSocket(cfd), 0, cip);
      }
  }



- 将文件描述符封装成conn对象

  - 将已连接socket文件描述符封装成connection对象
  - 初始化connection对象针对socket的一些方法（accept，read，write等）

  ```c
  connection *connCreateAcceptedSocket(int fd) {
      // 创建一个链接
      connection *conn = connCreateSocket();
      conn->fd = fd;
      // 状态设置为accepting
      conn->state = CONN_STATE_ACCEPTING;
      return conn;
  }
  
  connection *connCreateSocket() {
      connection *conn = zcalloc(sizeof(connection));
      conn->type = &CT_Socket;
      conn->fd = -1;
  
      return conn;
  }
  
  
  ConnectionType CT_Socket = {
      .ae_handler = connSocketEventHandler,
      .close = connSocketClose,
      .write = connSocketWrite,
      .read = connSocketRead,
      .accept = connSocketAccept,
      .connect = connSocketConnect,
      .set_write_handler = connSocketSetWriteHandler,
      .set_read_handler = connSocketSetReadHandler,
      .get_last_error = connSocketGetLastError,
      .blocking_connect = connSocketBlockingConnect,
      .sync_write = connSocketSyncWrite,
      .sync_read = connSocketSyncRead,
      .sync_readline = connSocketSyncReadLine,
      .get_type = connSocketGetType
  };
  ```



- 处理接收到的连接

  - 创建一个client对象，作为conn的私有数据
  - 设置conn的一些属性（将对应的socket设置成非阻塞、设置链接的读处理函数（readQueryFromClient）等）
  - 将对应的socket添加到epoll中，并监听socket的可读事件，设置该socket可读事件的处理函数为conn.ae_handler也就是connSocketEventHandler函数
  - 将client添加到server.clients中

  

  ```c
  static void acceptCommonHandler(connection *conn, int flags, char *ip) {
      client *c;
      char conninfo[100];
      UNUSED(ip);
  
    	// 检查链接状态是不是接受中
      if (connGetState(conn) != CONN_STATE_ACCEPTING) {
          serverLog(LL_VERBOSE,
                    "Accepted client connection in error state: %s (conn: %s)",
                    connGetLastError(conn),
                    connGetInfo(conn, conninfo, sizeof(conninfo)));
          connClose(conn);
          return;
      }
  
      ……
  
      // 1. 设置conn非阻塞
      // 2. 设置conn不延迟发包
      // 3. 设置读处理函数 readQueryFromClient
      // 4. 初始化client结构体
      // 5. 文件描述符对应的读处理函数为 connSocketSetReadHandler
      if ((c = createClient(conn)) == NULL) {
          //创建一个客户端
          serverLog(LL_WARNING,
                    "Error registering fd event for the new client: %s (conn: %s)",
                    connGetLastError(conn),
                    connGetInfo(conn, conninfo, sizeof(conninfo)));
          connClose(conn); /* May be already closed, just ignore errors */
          return;
      }
  
      
      c->flags |= flags;
  
      //设置连接状态为已连接，调用 clientAcceptHandler
    	// 不用关注这块
      if (connAccept(conn, clientAcceptHandler) == C_ERR) {
          char conninfo[100];
          if (connGetState(conn) == CONN_STATE_ERROR)
              serverLog(LL_WARNING,
                        "Error accepting a client connection: %s (conn: %s)",
                        connGetLastError(conn), connGetInfo(conn, conninfo, sizeof(conninfo)));
          freeClient(connGetPrivateData(conn));
          return;
      }
  }
  
  
  
  client *createClient(connection *conn) {
      client *c = zmalloc(sizeof(client));
  
      ……
      
      if (conn) {
          //socket 非阻塞
          connNonBlock(conn);
          // socket 不延迟发送包
          connEnableTcpNoDelay(conn);
          // 长链接
          if (server.tcpkeepalive)
              connKeepAlive(conn, server.tcpkeepalive);
          // 设置conn的读处理函数
          connSetReadHandler(conn, readQueryFromClient);
          // 设置conn的私有数据
          connSetPrivateData(conn, c);
      }
  
      selectDb(c, 0); // 选择0数据库
    
      ……
    
      // 将客户端添加到全局变量中，关联到server中
      if (conn) linkClient(c);
    
      // 初始化mstate
      initClientMultiState(c);
      return c;
  }
  ```

  

  - 将conn对应的socket添加到epoll

    - 设置该链接的读处理函数
    - 将socket添加到epoll中，监听可读事件，设置读处理函数为 connSocketEventHandler

    ```c
    static inline int connSetReadHandler(connection *conn, ConnectionCallbackFunc func) {
      	// 调用的是connSocketSetReadHandler，创建conn的时候初始化好的
        return conn->type->set_read_handler(conn, func);
    }
    
    static int connSocketSetReadHandler(connection *conn, ConnectionCallbackFunc func) {
        if (func == conn->read_handler) return C_OK;
    		
      	// 设置该连接的读处理函数
        conn->read_handler = func;
        // 同时将文件描述符添加到epoll中，该节点的处理函数是 connSocketEventHandler
        if (!conn->read_handler)
            aeDeleteFileEvent(server.el, conn->fd,AE_READABLE);
        else if (aeCreateFileEvent(server.el, conn->fd,
                                   AE_READABLE, conn->type->ae_handler, conn) == AE_ERR)
            return C_ERR;
        return C_OK;
    }
    ```

针对监听socket的可读事件处理就结束了



### 处理epoll上的事件（针对已连接socket）

监听socket会创建已连接socket，之后把socket添加到eventLoop->events 和 epoll中，这样在下一次aeProcessEvents函数调用的时候（也就是下一次循环，就可以处理已连接socket的事件了），当客户端建立连接后，向redis发送请求命令，会触发已连接socket的可读事件

```c
int aeProcessEvents(aeEventLoop *eventLoop, int flags) {
    int processed = 0, numevents;

    // 命中第一个条件，直接进入
    if (eventLoop->maxfd != -1 ||
        ((flags & AE_TIME_EVENTS) && !(flags & AE_DONT_WAIT))) {
        int j;
        
      	……

        if (eventLoop->beforesleep != NULL && flags & AE_CALL_BEFORE_SLEEP)
            eventLoop->beforesleep(eventLoop);

        // 返回事件的数量
        numevents = aeApiPoll(eventLoop, tvp);

        /* 回调事件 After sleep callback. */
        if (eventLoop->aftersleep != NULL && flags & AE_CALL_AFTER_SLEEP)
            eventLoop->aftersleep(eventLoop);

        for (j = 0; j < numevents; j++) {
            //拿到事件
            aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];
            int mask = eventLoop->fired[j].mask;
            int fd = eventLoop->fired[j].fd;
            int fired = 0;

          	//调用反转，忽略，这里不会有这种情况
            int invert = fe->mask & AE_BARRIER;

            if (!invert && fe->mask & mask & AE_READABLE) {
                //有读事件,调用读处理函数
                fe->rfileProc(eventLoop, fd, fe->clientData, mask);
                fired++;
                // 刷新一下，events的尺寸可能调整
                fe = &eventLoop->events[fd];
            }

            ……

            processed++;
        }
    }
    ……

    return processed;
}
```

#### 处理epoll上的事件

针对已连接socket来说，`fe->rfileProc(eventLoop, fd, fe->clientData, mask);` 会调用connSocketEventHandler函数

- 调用conn上注册的read_handler，也就是readQueryFromClient函数

  ```c
  static void connSocketEventHandler(struct aeEventLoop *el, int fd, void *clientData, int mask) {
      
    	……	
    
      if (invert && call_read) {
          if (!callHandler(conn, conn->read_handler)) return;
      }
  }
  ```

- readQueryFromClient函数

  - 从socket中读取客户端命令
  - 解析客户端命令保存到client中
  - 然后处理客户端命令
  
  ```c
  void readQueryFromClient(connection *conn) {
      client *c = connGetPrivateData(conn);
      int nread, readlen;
      size_t qblen;
  
    	// 多线程模型的处理
      if (postponeClientRead(c)) return;
  
    	//更新读次数
      server.stat_total_reads_processed++;
  
      readlen = PROTO_IOBUF_LEN;
      
      ……
  
      //获取缓冲区的长度
      qblen = sdslen(c->querybuf);
      //更新querybuf的峰值
      if (c->querybuf_peak < qblen) c->querybuf_peak = qblen;
      // 扩大字符串的可用空间
      c->querybuf = sdsMakeRoomFor(c->querybuf, readlen);
      // 从文件描述符获取数据
      nread = connRead(c->conn, c->querybuf + qblen, readlen);
      ……
  
      // 更新缓冲区字符串的大小
      sdsIncrLen(c->querybuf, nread);
      c->lastinteraction = server.unixtime; // 最后一次交互的时间
    
      ……
    	
      // 处理读取的客户端命令
    	processInputBuffer(c);
  }
  
  
  
  void processInputBuffer(client *c) {
      // 处理客户端缓存数据
      /* Keep processing while there is something in the input buffer */
      //不断处理缓冲区中的数据
      while (c->qb_pos < sdslen(c->querybuf)) {
          
        	……
  
          if (!c->reqtype) {
            	// redis的协议，发送命令会命中到这里
              if (c->querybuf[c->qb_pos] == '*') {
                  c->reqtype = PROTO_REQ_MULTIBULK;
              } else {
                  c->reqtype = PROTO_REQ_INLINE;
              }
          }
  
          if (c->reqtype == PROTO_REQ_INLINE) {
              ……
          } else if (c->reqtype == PROTO_REQ_MULTIBULK) {
            	// 这里会解析命令，然后保存到client中
              if (processMultibulkBuffer(c) != C_OK) break;
          } else {
              serverPanic("Unknown request type");
          }
  
          /* Multibulk processing could see a <= 0 length. */
          if (c->argc == 0) {
              resetClient(c);
          } else {
              ……
  						
              // 执行命令
              if (processCommandAndResetClient(c) == C_ERR) {
                  return;
              }
          }
      }
  
      /* Trim to pos */
      // 更新缓存的位置
      if (c->qb_pos) {
          // 截取pb_pos后面的内容 到 querybuf前面，更新qb_pos的值
          sdsrange(c->querybuf, c->qb_pos, -1);
          c->qb_pos = 0;
      }
  }
  ```
  
  - 处理客户端命令
  
    - 根据解析出来的命令，在哈希表中找到这个命令对应的函数
    - 调用call函数，执行这条命令
  
    ```c
    int processCommandAndResetClient(client *c) {
        int deadclient = 0;
        server.current_client = c;
      	// 执行命令
        if (processCommand(c) == C_OK) {
            // 主从同步相关，不管
            commandProcessed(c);
        }
        if (server.current_client == NULL) deadclient = 1;
        server.current_client = NULL;
      	// 返回C_OK
        return deadclient ? C_ERR : C_OK;
    }
    
    
    int processCommand(client *c) {
        ……
    
        // 从字典中拿到操作命令函数
        c->cmd = c->lastcmd = lookupCommand(c->argv[0]->ptr);
        // 找不到命令
        if (!c->cmd) {
            ……
        } else if ((c->cmd->arity > 0 && c->cmd->arity != c->argc) ||
                   (c->argc < -c->cmd->arity)) {
            // 检查命令需要的参数
            rejectCommandFormat(c, "wrong number of arguments for '%s' command",
                                c->cmd->name);
            return C_OK;
        }
    
      	// 设置该命令的相关标识
        int is_write_command = (c->cmd->flags & CMD_WRITE) ||
                               (c->cmd->proc == execCommand && (c->mstate.cmd_flags & CMD_WRITE));
        int is_denyoom_command = (c->cmd->flags & CMD_DENYOOM) ||
                                 (c->cmd->proc == execCommand && (c->mstate.cmd_flags & CMD_DENYOOM));
        int is_denystale_command = !(c->cmd->flags & CMD_STALE) ||
                                   (c->cmd->proc == execCommand && (c->mstate.cmd_inv_flags & CMD_STALE));
        int is_denyloading_command = !(c->cmd->flags & CMD_LOADING) ||
                                     (c->cmd->proc == execCommand && (c->mstate.cmd_inv_flags & CMD_LOADING));
    
        ……
    
        if (c->flags & CLIENT_MULTI &&
            c->cmd->proc != execCommand && c->cmd->proc != discardCommand &&
            c->cmd->proc != multiCommand && c->cmd->proc != watchCommand) {
            queueMultiCommand(c);
            addReply(c, shared.queued);
        } else {
            // 执行这条命令
            call(c,CMD_CALL_FULL);
            c->woff = server.master_repl_offset;
            if (listLength(server.ready_keys))
                handleClientsBlockedOnKeys();
        }
        return C_OK;
    }
    ```
  
  - call函数，找个具体的命令看下做了什么操作（set命令）
  
    ```c
    void call(client *c, int flags) {
        long long dirty;
        ustime_t start, duration;
        int client_old_flags = c->flags;
        struct redisCommand *real_cmd = c->cmd;
    
        ……
    
        c->flags &= ~(CLIENT_FORCE_AOF | CLIENT_FORCE_REPL | CLIENT_PREVENT_PROP);
        ……
    
        // 统计调用命令的耗时
        start = server.ustime;
        // 调用函数，执行命令
        c->cmd->proc(c);
        duration = ustime() - start;
        dirty = server.dirty - dirty;
      
        ……
          
    }
    ```
  
  - setCommand
  
    - 执行具体的set命令
    - 将响应值写入client
  
    ```c
    /* SET key value [NX] [XX] [KEEPTTL] [EX <seconds>] [PX <milliseconds>] */
    void setCommand(client *c) {
        int j;
        robj *expire = NULL;
        int unit = UNIT_SECONDS;
        int flags = OBJ_SET_NO_FLAGS;
    
      	// 解析具体的set操作
        for (j = 3; j < c->argc; j++) {
            char *a = c->argv[j]->ptr;
            robj *next = (j == c->argc-1) ? NULL : c->argv[j+1];
    
            if ((a[0] == 'n' || a[0] == 'N') &&
                (a[1] == 'x' || a[1] == 'X') && a[2] == '\0' &&
                !(flags & OBJ_SET_XX))
            {
                flags |= OBJ_SET_NX;
            } else if ((a[0] == 'x' || a[0] == 'X') &&
                       (a[1] == 'x' || a[1] == 'X') && a[2] == '\0' &&
                       !(flags & OBJ_SET_NX))
            {
                flags |= OBJ_SET_XX;
            } else if (!strcasecmp(c->argv[j]->ptr,"KEEPTTL") &&
                       !(flags & OBJ_SET_EX) && !(flags & OBJ_SET_PX))
            {
                flags |= OBJ_SET_KEEPTTL;
            } else if ((a[0] == 'e' || a[0] == 'E') &&
                       (a[1] == 'x' || a[1] == 'X') && a[2] == '\0' &&
                       !(flags & OBJ_SET_KEEPTTL) &&
                       !(flags & OBJ_SET_PX) && next)
            {
                flags |= OBJ_SET_EX;
                unit = UNIT_SECONDS;
                expire = next;
                j++;
            } else if ((a[0] == 'p' || a[0] == 'P') &&
                       (a[1] == 'x' || a[1] == 'X') && a[2] == '\0' &&
                       !(flags & OBJ_SET_KEEPTTL) &&
                       !(flags & OBJ_SET_EX) && next)
            {
                flags |= OBJ_SET_PX;
                unit = UNIT_MILLISECONDS;
                expire = next;
                j++;
            } else {
                addReply(c,shared.syntaxerr);
                return;
            }
        }
    		
      	// value
        c->argv[2] = tryObjectEncoding(c->argv[2]);
      	// 执行命令
        setGenericCommand(c,flags,c->argv[1],c->argv[2],expire,unit,NULL,NULL);
    }
    
    
    void setGenericCommand(client *c, int flags, robj *key, robj *val, robj *expire, int unit, robj *ok_reply, robj *abort_reply) {
        long long milliseconds = 0; /* initialized to avoid any harmness warning */
    
        ……
        // 执行set操作
        genericSetKey(c,c->db,key,val,flags & OBJ_SET_KEEPTTL,1);
        ……
        // 添加返回值给客户端（并不会立即写回客户端）
        addReply(c, ok_reply ? ok_reply : shared.ok);
    }
    
    ```
  
  - addReply
  
    - 将client添加到`server.clients_pending_write`，准备好响应客户端，但不会立即响应客户端，将client设置为`CLIENT_PENDING_WRITE`
    - 先把响应值添加到client的buf中
    - buf满了，将响应值添加到链表中
  
    ```c
    void addReply(client *c, robj *obj) {
        // 准备写入数据
        if (prepareClientToWrite(c) != C_OK) return;
    
        // 不同的编码值
        if (sdsEncodedObject(obj)) {
            // buf不能添加，添加到队列中
            if (_addReplyToBuffer(c, obj->ptr, sdslen(obj->ptr)) != C_OK)
                // 添加到队列
                _addReplyProtoToList(c, obj->ptr, sdslen(obj->ptr));
        } else if (obj->encoding == OBJ_ENCODING_INT) {
            /* For integer encoded strings we just convert it into a string
             * using our optimized function, and attach the resulting string
             * to the output buffer. */
            char buf[32];
            size_t len = ll2string(buf, sizeof(buf), (long) obj->ptr);
            if (_addReplyToBuffer(c, buf, len) != C_OK)
                _addReplyProtoToList(c, buf, len);
        } else {
            serverPanic("Wrong obj->encoding in addReply()");
        }
    }
    ```
  

对于客户端命令的处理流程如上。



### beforeSleep

调用`handleClientsWithPendingWritesUsingThreads`去响应客户端

```c
void beforeSleep(struct aeEventLoop *eventLoop) {
    ……

    // 多线程处理客户端请求，这里是多线程模型处理的逻辑
    handleClientsWithPendingReadsUsingThreads();

    ……

    // 将响应值写回客户端，将响应值返回给客户端
    handleClientsWithPendingWritesUsingThreads();

    // 异步关闭客户端
    freeClientsInAsyncFreeQueue();

    ……
}
```

- handleClientsWithPendingWritesUsingThreads

  - 遍历每个客户端，将响应写回客户端

  ```c
  int handleClientsWithPendingWritesUsingThreads(void) {
    	// 待响应的客户端个数
      int processed = listLength(server.clients_pending_write);
    	// 没有客户端响应直接返回
      if (processed == 0) return 0; /* Return ASAP if there are no clients. */
  
      // 单线程处理
      if (server.io_threads_num == 1 || stopThreadedIOIfNeeded()) {
          // 单线程处理响应
          return handleClientsWithPendingWrites();
      }
  
     	……
  
      return processed;
  }
  
  
  int handleClientsWithPendingWrites(void) {
      listIter li;
      listNode *ln;
      // 客户端个数
      int processed = listLength(server.clients_pending_write);
  
      // 创建迭代器
      listRewind(server.clients_pending_write, &li);
      // 取出链表上的值
      while ((ln = listNext(&li))) {
          // 拿到client
          client *c = listNodeValue(ln);
        
          // 去掉CLIENT_PENDING_WRITE标识
          c->flags &= ~CLIENT_PENDING_WRITE;
          
        	……
  
          // 把数据写回到客户端
          if (writeToClient(c, 0) == C_ERR) continue;
        
        	// socket的send_buf满了，注册一个可写事件，在可写的时候通知
          if (clientHasPendingReplies(c)) {
              int ae_barrier = 0;
              
            	……
  
              // 设置写处理函数
              if (connSetWriteHandlerWithBarrier(c->conn, sendReplyToClient, ae_barrier) == C_ERR) {
                  freeClientAsync(c);
              }
          }
  
          ……
      }
      return processed;
  }
  ```

#### 注册写处理函数

正常来说会将要响应客户端的缓存数据全部发给客户端，但是也可能出现socket的send_buf满的情况，这个时候客户端会有遗留的数据，需要针对这个socket向注册一个可写事件，当socket可写时，可以通知客户端来处理。

- 调用`connSetWriteHandlerWithBarrier`函数来注册这个conn的写处理函数，注册的函数为`sendReplyToClient`
- `connSetWriteHandlerWithBarrier函数`会调用`connSocketSetWriteHandler函数`（conn初始化时配置好的函数）
- `connSocketSetWriteHandler`函数会向epoll针对这个socket注册一个可写事件，处理函数为`connSocketEventHandler`（同可读事件处理一致）
- 这样，当可写事件触发时，就会调用到`sendReplyToClient`函数，完成该链接对客户端的响应

```c
int handleClientsWithPendingWrites(void) {
    listIter li;
    listNode *ln;
    // 客户端个数
    int processed = listLength(server.clients_pending_write);

    // 创建迭代器
    listRewind(server.clients_pending_write, &li);
    // 取出链表上的值
    while ((ln = listNext(&li))) {
        ……
      
      	// socket的send_buf满了，注册一个可写事件，在可写的时候通知
        if (clientHasPendingReplies(c)) {
            int ae_barrier = 0;
            
          	……

            // 设置写处理函数
            if (connSetWriteHandlerWithBarrier(c->conn, sendReplyToClient, ae_barrier) == C_ERR) {
                freeClientAsync(c);
            }
        }

        ……
    }
    return processed;
}


static inline int connSetWriteHandlerWithBarrier(connection *conn, ConnectionCallbackFunc func, int barrier) {
    return conn->type->set_write_handler(conn, func, barrier);
}


static int connSocketSetWriteHandler(connection *conn, ConnectionCallbackFunc func, int barrier) {
    if (func == conn->write_handler) return C_OK;

    conn->write_handler = func;
    if (barrier)
        conn->flags |= CONN_FLAG_WRITE_BARRIER;
    else
        conn->flags &= ~CONN_FLAG_WRITE_BARRIER;
    if (!conn->write_handler)
        aeDeleteFileEvent(server.el, conn->fd,AE_WRITABLE);
    else if (aeCreateFileEvent(server.el, conn->fd,AE_WRITABLE,
                               conn->type->ae_handler, conn) == AE_ERR)
        return C_ERR;
    return C_OK;
}
```





从客户端到服务端请求的处理整个流程如上。整个过程都是单线程处理。



# 多线程

启动多线程需要在配置文件配置

```conf
io-threads 4
io-threads-do-reads yes
```



## 相关主流程

多线程的主流程跟单线程的差不多，但是多了一个多线程的创建

```c
int main(int argc, char **argv) {
    ……
		
    //初始化server结构体，包括文件描述符的创建
    initServer();
    ……

    if (!server.sentinel_mode) {
        ……
          
        //初始化server，创建后台线程,创建IO线程
        InitServerLast();
      
        ……
    } else {
        ……
    }

    ……

    // 跑主函数
    aeMain(server.el);
    
  	……
    
    return 0;
}
```



## IO线程的创建

`InitServerLast`会根据配置的线程数来启动多个线程，线程对应的处理函数为`IOThreadMain`

多线程由`handleClientsWithPendingWritesUsingThreads`函数来激活

- 程序第一次处理客户端请求的时候，还是主线程单线程来处理
- 当要响应客户端的时候，会激活多线程并多线程响应客户端

```c
void InitServerLast() {
    ……
    
    //初始化线程IO，redis的多线程模型就是在这里创建的
    initThreadedIO();
  	
    ……
}


void initThreadedIO(void) {
    server.io_threads_active = 0;

    ……
    if (server.io_threads_num == 1) return;

    // 最大128个线程，超过会退出程序
    if (server.io_threads_num > IO_THREADS_MAX_NUM) {
        serverLog(LL_WARNING, "Fatal: too many I/O threads configured. "
                  "The maximum number is %d.", IO_THREADS_MAX_NUM);
        exit(1);
    }

    // 初始化线程
    for (int i = 0; i < server.io_threads_num; i++) {
        /* Things we do for all the threads including the main thread. */
        io_threads_list[i] = listCreate();
      	// aeMain的线程在负责
        if (i == 0) continue;

        // 初始化线程
        pthread_t tid;
        pthread_mutex_init(&io_threads_mutex[i],NULL);
        io_threads_pending[i] = 0;
        // 加锁
        pthread_mutex_lock(&io_threads_mutex[i]); /* Thread will be stopped. */
        // 创建线程
        if (pthread_create(&tid,NULL, IOThreadMain, (void *) (long) i) != 0) {
            serverLog(LL_WARNING, "Fatal: Can't initialize IO thread.");
            exit(1);
        }
        // 线程id
        io_threads[i] = tid;
    }
}
```



## 事件循环处理

### 处理epoll上的事件（针对监听socket）

即使开启了多线程，redis对监听socket的操作和单线程是一样的，都是由主线程来处理。



### 处理epoll上的事件（针对已连接socket）

前置条件，多线程被唤醒激活，即server.io_threads_active=1。

第一次处理客户端的请求时，还没有激活，流程与单线程一样。当第一次响应客户端时，线程就会被激活，然后处理客户端的请求也用多线程

```c
void beforeSleep(struct aeEventLoop *eventLoop) {
  
    ……

    // 将响应值写回客户端
    handleClientsWithPendingWritesUsingThreads();
  
  	……
}

int handleClientsWithPendingWritesUsingThreads(void) {
    ……
    if (!server.io_threads_active) startThreadedIO();

    ……

    return processed;
}


void startThreadedIO(void) {
    ……
    serverAssert(server.io_threads_active == 0);
    // 激活其他线程
    for (int j = 1; j < server.io_threads_num; j++)
        pthread_mutex_unlock(&io_threads_mutex[j]);
    server.io_threads_active = 1;
}
```



#### 处理epoll上的事件

从上面单线程的处理分析，处理已连接socket事件会调用`readQueryFromClient`函数

- 可读事件触发后conn第一次调用`postponeClientRead`后，`readQueryFromClient`会被拦截返回
- c->flags 会被设置成`CLIENT_PENDING_READ`，将当前client的处理放到`server.clients_pending_read`中

```c
void readQueryFromClient(connection *conn) {
    client *c = connGetPrivateData(conn);
    int nread, readlen;
    size_t qblen;

    // 多线程处理
    if (postponeClientRead(c)) return;
  
  	……
}


int postponeClientRead(client *c) {
    if (server.io_threads_active &&
        server.io_threads_do_reads &&
        !clientsArePaused() &&
        !ProcessingEventsWhileBlocked &&
        !(c->flags & (CLIENT_MASTER | CLIENT_SLAVE | CLIENT_PENDING_READ))) {
        // 设置成CLIENT_PENDING_READ，下次调用就不会进入这个分支了
        c->flags |= CLIENT_PENDING_READ;
        listAddNodeHead(server.clients_pending_read, c);
        return 1;
    } else {
        return 0;
    }
}
```



### beforeSleep

在一次循环中`beforeSleep`是在处理epoll事件之前调用的，所以本次调用是在下一次循环

- 多线程处理客户端请求，调用`handleClientsWithPendingReadsUsingThreads`函数

```c
void beforeSleep(struct aeEventLoop *eventLoop) {
    ……

    // 多线程处理客户端请求
    handleClientsWithPendingReadsUsingThreads();

    ……

    // 将响应值写回客户端
    handleClientsWithPendingWritesUsingThreads();

    // 异步关闭客户端
    freeClientsInAsyncFreeQueue();

  	……
}
```

- `handleClientsWithPendingReadsUsingThreads`函数

  - 将客户端请求从`server.clients_pending_read`中读取出来
  - 将不同的客户端请求分发到不同的`io_threads_list`中，以便多线程能处理客户端请求
  - `io_threads_pending`被设置为客户端数，这个时候会通知IO线程来处理请求
  - 主线程也负责一部分的请求处理
  - 最后等待所有线程完成工作，才往后继续处理

  ```c
  int handleClientsWithPendingReadsUsingThreads(void) {
      ……
      listIter li;
      listNode *ln;
      listRewind(server.clients_pending_read, &li);
      int item_id = 0;
      // 将客户端请求的处理分配到不同的线程
      while ((ln = listNext(&li))) {
          // 遍历客户端,并没有从这个队列中删除
          client *c = listNodeValue(ln);
          // 分配到某个线程
          int target_id = item_id % server.io_threads_num;
          // 放到多线程对应的队列上
          listAddNodeTail(io_threads_list[target_id], c);
          item_id++;
      }
  
      // 设置为可读，多线程开始处理
      io_threads_op = IO_THREADS_OP_READ;
      for (int j = 1; j < server.io_threads_num; j++) {
          int count = listLength(io_threads_list[j]);
        	// io_threads_pending设置为大于0的值，其他IO线程才能开始工作
          io_threads_pending[j] = count;
      }
  
      // 主线程自己负责一部分请求
      listRewind(io_threads_list[0], &li);
      while ((ln = listNext(&li))) {
          client *c = listNodeValue(ln);
          // 处理客户端请求,解析协议，将(一条)命令保存到client，并将client的标识设置为CLIENT_PENDING_COMMAND
          readQueryFromClient(c->conn);
      }
      listEmpty(io_threads_list[0]);
  
      // 这里等待所有线程的工作都完成了
      while (1) {
          // 等待所有线程完成任务
          unsigned long pending = 0;
          for (int j = 1; j < server.io_threads_num; j++)
              pending += io_threads_pending[j];
          if (pending == 0) break;
      }
      ……
  
      return processed;
  }
  ```

- 看IO线程是如何处理请求的，IO线程的运行函数是`IOThreadMain`

  - `io_threads_pending`被设置时，该线程开始工作
  - 将`io_threads_list`对应线程的任务取出来，调用`readQueryFromClient`

  ```c
  void *IOThreadMain(void *myid) {
      /* The ID is the thread number (from 0 to server.iothreads_num-1), and is
       * used by the thread to just manipulate a single sub-array of clients. */
      long id = (unsigned long) myid;
      char thdname[16];
  
      ……
  
      while (1) {
          /* Wait for start */
          // 等待线程启动
          for (int j = 0; j < 1000000; j++) {
              if (io_threads_pending[id] != 0) break;
          }
  
          // io_threads_pending[id]被设置时开始工作，执行下面的逻辑
          if (io_threads_pending[id] == 0) {
              // 当多线程没被激活时，这里加锁会被阻塞 server.io_threads_active = 0
              // 当多线程被激活时，这里会加锁又立马解锁 server.io_threads_active = 1
              pthread_mutex_lock(&io_threads_mutex[id]);
              pthread_mutex_unlock(&io_threads_mutex[id]);
              continue;
          }
  
          ……
          
          listIter li;
          listNode *ln;
          // 将li链接到链表中，迭代器
          listRewind(io_threads_list[id], &li);
          // 遍历链表
          while ((ln = listNext(&li))) {
              // 取出保存的值，一个客户端请求
              client *c = listNodeValue(ln);
              if (io_threads_op == IO_THREADS_OP_WRITE) {
                  writeToClient(c, 0); // 处理写请求
              } else if (io_threads_op == IO_THREADS_OP_READ) {
                  // 处理读请求
                  readQueryFromClient(c->conn);
              } else {
                  serverPanic("io_threads_op value is unknown");
              }
          }
          listEmpty(io_threads_list[id]); // 清空链表
          io_threads_pending[id] = 0; // 重置状态
  
          ……
      }
  }
  ```

- 多线程`readQueryFromClient`函数的处理，在将client添加进`server.clients_pending_read`时，c->flags 已经被设置成`CLIENT_PENDING_READ`

  - 将请求从socket中读到client的buf中，然后调用`processInputBuffer`函数

  ```c
  void readQueryFromClient(connection *conn) {
      
      // 多线程处理
    	// c->flags = CLIENT_PENDING_READ，会跳过这个拦截，执行下面的代码
      if (postponeClientRead(c)) return;
  
      ……
  
      // 读取的长度
      readlen = PROTO_IOBUF_LEN;
      
      if (c->reqtype == PROTO_REQ_MULTIBULK && c->multibulklen && c->bulklen != -1
          && c->bulklen >= PROTO_MBULK_BIG_ARG) {
          ssize_t remaining = (size_t) (c->bulklen + 2) - sdslen(c->querybuf);
  
          /* Note that the 'remaining' variable may be zero in some edge case,
           * for example once we resume a blocked client after CLIENT PAUSE. */
          if (remaining > 0 && remaining < readlen) readlen = remaining;
      }
  
      //获取缓冲区的长度
      qblen = sdslen(c->querybuf);
      //更新querybuf的峰值
      if (c->querybuf_peak < qblen) c->querybuf_peak = qblen;
      // 扩大字符串的可用空间
      c->querybuf = sdsMakeRoomFor(c->querybuf, readlen);
      // 从文件描述符获取数据，从free空间读取
      nread = connRead(c->conn, c->querybuf + qblen, readlen);
      ……
  
      // 更新缓冲区字符串的大小
      sdsIncrLen(c->querybuf, nread);
      c->lastinteraction = server.unixtime; // 最后一次交互的时间
      if (c->flags & CLIENT_MASTER) c->read_reploff += nread; //主从同步相关
      server.stat_net_input_bytes += nread;
      //检查缓存长度是否大于限制
      if (sdslen(c->querybuf) > server.client_max_querybuf_len) {
          sds ci = catClientInfoString(sdsempty(), c), bytes = sdsempty();
  
          bytes = sdscatrepr(bytes, c->querybuf, 64);
          serverLog(LL_WARNING, "Closing client that reached max query buffer length: %s (qbuf initial bytes: %s)", ci,
                    bytes);
          sdsfree(ci);
          sdsfree(bytes);
          freeClientAsync(c);
          return;
      }
  
      
      processInputBuffer(c);
  }
  ```

- `processInputBuffer`函数

  - 解析客户端命令，保存到c->argv中
  - 将c-flags设置成`CLIENT_PENDING_COMMAND`

  ```c
  void processInputBuffer(client *c) {
      // 处理客户端缓存数据
      /* Keep processing while there is something in the input buffer */
      //不断处理缓冲区中的数据
      while (c->qb_pos < sdslen(c->querybuf)) {
          
        	……
  
          if (c->reqtype == PROTO_REQ_INLINE) {
              ……
          } else if (c->reqtype == PROTO_REQ_MULTIBULK) {
              if (processMultibulkBuffer(c) != C_OK) break;
          } else {
              serverPanic("Unknown request type");
          }
  
          /* Multibulk processing could see a <= 0 length. */
          if (c->argc == 0) {
              resetClient(c);
          } else {
              /* If we are in the context of an I/O thread, we can't really
               * execute the command here. All we can do is to flag the client
               * as one that needs to process the command. */
              if (c->flags & CLIENT_PENDING_READ) {
                  c->flags |= CLIENT_PENDING_COMMAND;
                  break;
              }
  
              /* We are finally ready to execute the command. */
              // 执行这条命令
              if (processCommandAndResetClient(c) == C_ERR) {
                  /* If the client is no longer valid, we avoid exiting this
                   * loop and trimming the client buffer later. So we return
                   * ASAP in that case. */
                  return;
              }
          }
      }
  
      /* Trim to pos */
      // 更新缓存的位置
      if (c->qb_pos) {
          // 截取pb_pos后面的内容 到 querybuf前面，更新qb_pos的值
          sdsrange(c->querybuf, c->qb_pos, -1);
          c->qb_pos = 0;
      }
  }
  ```



到这里多线程的处理就完成了，回到主线程

#### 多线程处理请求后，主线程的处理

- 将这批客户端请求一个个拿出来，然后执行命令
  - 去掉c->flags的`CLIENT_PENDING_READ`标识，去掉c->flags的`CLIENT_PENDING_COMMAND`标识
  - 调用`processPendingCommandsAndResetClient`，这里会调用`processCommandAndResetClient`，接着的过程就和单线程的一模一样了
  - 如果客户端还有一些缓存没处理，会尝试处理剩余的请求，调用`processInputBuffer`函数，和单线程的一样

```c
int handleClientsWithPendingReadsUsingThreads(void) {
    ……
    
    // 这里等待所有线程的工作都完成了
    while (1) {
        // 等待所有线程完成任务
        unsigned long pending = 0;
        for (int j = 1; j < server.io_threads_num; j++)
            pending += io_threads_pending[j];
        if (pending == 0) break;
    }
    if (tio_debug) printf("I/O READ All threads finshed\n");

    // 处理这批客户端请求
    while (listLength(server.clients_pending_read)) {
        // 处理这批客户端请求,主线程处理（顺序执行）
        // 拿出队头
        ln = listFirst(server.clients_pending_read);
        client *c = listNodeValue(ln);
        // 去掉CLIENT_PENDING_READ标识
        c->flags &= ~CLIENT_PENDING_READ;
        // 从队列删除队头
        listDelNode(server.clients_pending_read, ln);
      
        ……

        // 执行客户端的命令
        if (processPendingCommandsAndResetClient(c) == C_ERR) {
            continue;
        }

        // 继续执行客户端剩余缓存的请求
        processInputBuffer(c);

        
        ……
    }

   	……

    return processed;
}
```



#### 多线程对响应的处理

```c
int handleClientsWithPendingWritesUsingThreads(void) {
    int processed = listLength(server.clients_pending_write);
    if (processed == 0) return 0; /* Return ASAP if there are no clients. */

    ……

    /* Distribute the clients across N different lists. */
    listIter li;
    listNode *ln;
    listRewind(server.clients_pending_write, &li);
    int item_id = 0;
    while ((ln = listNext(&li))) {
        client *c = listNodeValue(ln);
        c->flags &= ~CLIENT_PENDING_WRITE;

        ……

        // 将客户单的响应分配给不同的线程
        int target_id = item_id % server.io_threads_num;
        listAddNodeTail(io_threads_list[target_id], c);
        item_id++;
    }

    // 通知对应的线程可以开始处理响应
    io_threads_op = IO_THREADS_OP_WRITE;
    for (int j = 1; j < server.io_threads_num; j++) {
        int count = listLength(io_threads_list[j]);
        io_threads_pending[j] = count;
    }

    // 主线程负责一部分工作
    listRewind(io_threads_list[0], &li);
    while ((ln = listNext(&li))) {
        client *c = listNodeValue(ln);
        writeToClient(c, 0);
    }
    listEmpty(io_threads_list[0]);

    // 等待所有线程完成工作
    while (1) {
        unsigned long pending = 0;
        for (int j = 1; j < server.io_threads_num; j++)
            pending += io_threads_pending[j];
        if (pending == 0) break;
    }
    if (tio_debug) printf("I/O WRITE All threads finshed\n");

    /* Run the list of clients again to install the write handler where
     * needed. */
    listRewind(server.clients_pending_write, &li);
    while ((ln = listNext(&li))) {
        client *c = listNodeValue(ln);

        // 如果客户端还有未处理的响应，在epoll上注册可写事件
        if (clientHasPendingReplies(c) &&
            connSetWriteHandler(c->conn, sendReplyToClient) == AE_ERR) {
            freeClientAsync(c);
        }
    }
    listEmpty(server.clients_pending_write);

    /* Update processed count on server */
    server.stat_io_writes_processed += processed;

    return processed;
}
```







# 总结

**所以，redis的多线程：（只是处理请求的解析和响应，命令的执行是由主线程单线程执行的）**

**对于客户端的请求，只是完成了从socket读取数据，再将数据解析成具体的命令。**

**对于客户端的响应，会将本次执行的命令给客户端的响应，分多个线程，然后去完全客户端的响应。**

**之后，具体的命令执行是由主线程执行的，也就是说redis的命令是单线程执行的**





# 函数

## 线程

```c
// pthread_t 创建线程后，线程ID会写入到这里
// pthread_attr_t 设置线程的属性，如栈大小等...
// __start_routine 线程函数
// __restrict 线程函数的参数
extern int pthread_create (pthread_t *__restrict __newthread,
			   const pthread_attr_t *__restrict __attr,
			   void *(*__start_routine) (void *),
			   void *__restrict __arg) __THROWNL __nonnull ((1, 3));
```

