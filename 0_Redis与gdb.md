# GDB & REdis
顺着bt去阅读

### 开启侦听过程
+ socket
+ bind
+ listen   
  

在源码中找到这个函数的位置，加上断点。比如找到 `listen`的所在的函數位置，
``` cpp
  (gdb) break anetListen
    Breakpoint 1 at 0x8040457: file anet.c, line 455.
```

然後重啓：
```cpp
    (gdb) run 
```
就会进入该断点位置：
```cpp
      Breakpoint 1, anetListen (err=0x8809e68 <server+680> "", s=6, sa=0x8c1e980, len=28, backlog=511)
      at anet.c:455
      455         if (bind(s,sa,len) == -1) {
      (gdb) 
```
bind绑定了本地的任何一个可用的地址。
```cpp
  for (p = servinfo; p != NULL; p = p->ai_next) {
      if ((s = socket(p->ai_family,p->ai_socktype,p->ai_protocol)) == -1)
          continue;

      if (af == AF_INET6 && anetV6Only(err,s) == ANET_ERR) goto error;
      if (anetSetReuseAddr(err,s) == ANET_ERR) goto error;
      if (anetListen(err,s,p->ai_addr,p->ai_addrlen,backlog) == ANET_ERR) s = ANET_ERR;
      goto end; //有一个可用的就退出
  }
```

值得学习。
#### epollFd 的创建
搜索 epoll_create

加上断点，重启就到了断点处，
```cpp
  #0  aeApiCreate (eventLoop=0x7ffffe20b480) at ae_epoll.c:40
  #1  0x000000000803e3b7 in aeCreateEventLoop (setsize=10128) at ae.c:80
  #2  0x00000000080474ec in initServer () at server.c:2780
  #3  0x000000000804da49 in main (argc=1, argv=0x7ffffffedf68) at server.c:5126
```

#### 把listenfd 挂在到 epollfd 上

搜索 `epoll_ctl`，定位到 `aeApiAddEvent`，加上断点

在 `anetListen`的输入参数中 `s` 就是 `epollfd`。

运行到`aeApiAddEvent`，查看`bt`

```
  (gdb) bt
  #0  aeApiAddEvent (eventLoop=0x7ffffe20b480, fd=6, mask=1) at ae_epoll.c:73
  #1  0x000000000803e66f in aeCreateFileEvent (eventLoop=0x7ffffe20b480, fd=6, mask=1, proc=0x8059dce <acceptTcpHandler>, clientData=0x0)
      at ae.c:162
  #2  0x0000000008047b1e in initServer () at server.c:2885
  #3  0x000000000804da49 in main (argc=1, argv=0x7ffffffedf68) at server.c:5126
```


完成上述准备工作就进入epoll开始主循环。

listenfd 关注可读事件，接受新的连接，产生 clientfd --> 搜索accept-->位置：anetGenericAccept(...)，加上断点

删除之前的断点，加上新的这个断点
```cpp
  delete 
  break anetGenericAccept
```

启动一个客户端，此时就会触发刚才设置的断点
```cpp
  Thread 1 "redis-server" hit Breakpoint 5, anetGenericAccept (err=0x8809e68 <server+680> "", s=7, sa=0x7ffffffedc40, len=0x7ffffffedc28)
  at anet.c:548
  548             fd = accept(s,sa,len);
```

查看此时的bt
```cpp
  (gdb) bt
  #0  anetGenericAccept (err=0x8809e68 <server+680> "", s=7, sa=0x7ffffffedc40, len=0x7ffffffedc28) at anet.c:548
  #1  0x00000000080409e4 in anetTcpAccept (err=0x8809e68 <server+680> "", s=7, ip=0x7ffffffedd10 "`\337\376\377\377\177", ip_len=46, port=0x7ffffffedd04) at anet.c:566
  #2  0x0000000008059e20 in acceptTcpHandler (el=0x7ffffe20b480, fd=7, privdata=0x0, mask=1) at networking.c:955
  #3  0x000000000803f08f in aeProcessEvents (eventLoop=0x7ffffe20b480, flags=27) at ae.c:479
  #4  0x000000000803f2bd in aeMain (eventLoop=0x7ffffe20b480) at ae.c:539
  #5  0x000000000804dbea in main (argc=1, argv=0x7ffffffedf68) at server.c:5173
```

可以查看之前对于接受clientfd 时候的回调：
```cpp
  (gdb) f 3
  #3  0x000000000803f08f in aeProcessEvents (eventLoop=0x7ffffe20b480, flags=27) at ae.c:479
  warning: Source file is more recent than executable.
  479                     fe->rfileProc(eventLoop,fd,fe->clientData,mask);
```
这个 `fe->rfileProc` 回调调的是  `acceptTcpHandler`

那么，什么时候设置的这个回调函数呢？经过搜索发现在 `initServer` 中
```cpp
  for (j = 0; j < server.tlsfd_count; j++) {
  if (aeCreateFileEvent(server.el, server.tlsfd[j], AE_READABLE, acceptTLSHandler,NULL) == AE_ERR)
      {
          serverPanic(
              "Unrecoverable error creating server.tlsfd file event.");
      }
  }
```

这个指针的赋值`rfileProc`
```cpp
  int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask, aeFileProc *proc, void *clientData)
  {
      if (fd >= eventLoop->setsize) {
          errno = ERANGE;
          return AE_ERR;
      }
      aeFileEvent *fe = &eventLoop->events[fd];

      if (aeApiAddEvent(eventLoop, fd, mask) == -1)
          return AE_ERR;
      fe->mask |= mask;
      if (mask & AE_READABLE) fe->rfileProc = proc;
      if (mask & AE_WRITABLE) fe->wfileProc = proc;
      fe->clientData = clientData;
      if (fd > eventLoop->maxfd)
          eventLoop->maxfd = fd;
      return AE_OK;
  }
```

查看怎么运行到一个函数，就在这个函数处加上断点，触发这个断点后 `bt`，查看信息即可。

因此，listenfd 的处理函数就是 `acceptTcpHandler`

### 将clientFd 挂到 epllfd

因为redis是单线程，因此需要把他也挂在 listenfd 所在的 epollfd 上，而在redis中，所有的函数都是在 `aeApiAddEvent` 中完成的


`acceptCommonHandler(connCreateAcceptedSocket(cfd),0,cip);`函数在 
```cpp
  connection *connCreateAcceptedSocket(int fd) {
      connection *conn = connCreateSocket();
      conn->fd = fd;
      conn->state = CONN_STATE_ACCEPTING;
      return conn;
  }
```
创建一个 `conn` 对象，设置 `clintfd`

### redis 通信协议

`clientfd`的读取与发送数据： 


`clientfd` 的读取回调函数 `readQueryFromClient` 中处理协议
+ 先分配接受缓冲区
+ `connRead` --> `read`

解包过程，放在while循环里


redis 多线程处理


一个请求到来时候的惊群问题

当 io 繁忙的时候定时器不一定精确了，需要更新。redis对于耗时的任务开了个新的线程去执行

获取系统时间是系统调用，耗时
根据定时器的时间来确定 epoll_wait 的阻塞时间，使其不必阻塞太久