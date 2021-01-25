# EventLoop

EventLoop是REdis得以高效运行的关键部分。

## aeEventLoop

```cpp
  typedef struct aeEventLoop {
      int maxfd;  			 /* 当前注册的fd最大数 */
      int setsize; 			 /* 能注册的最大fd数 */
      long long timeEventNextId; 	
      time_t lastTime;     /* Used to detect system clock skew */ 
      aeFileEvent *events; /* 已经注册的事件， events 建立了一个映射关系：fd --> event*/
      aeFiredEvent *fired; /* 已经触发的事件，记录的是触发的fd，及其事件类型 */
      aeTimeEvent *timeEventHead;
      int stop;           // EventLoop 是否停止
      void* apidata;      /* This is used for polling API specific data ，这个用于存放和 epollfd 相关的数据*/
      aeBeforeSleepProc *beforesleep; // 在 epoll_wait 阻塞之前调用的函数
      aeBeforeSleepProc *aftersleep;  // 在 epoll_wait 唤醒之后调用的函数
      int flags;        // 设置的标志位
  } aeEventLoop;
```
+ `events`

  这是一个数组，在文件描述符`fd`和`fd`所关注的事件之间建立映射关系。

  ```cpp
  typedef struct aeFileEvent {
      int mask; 				// AE_NONE、AE_READABLE 、AE_WRITABLE、AE_BARRIER 
      aeFileProc *rfileProc;   // 可读事件的回调函数 
      aeFileProc *wfileProc;	 // 可写事件的回调函数
      void *clientData; 		 // clinet
  } aeFileEvent;
  
  ```

+ `fired`

    数组，记录了有事件触发的文件描述及其对应的类型

    ```cpp
    typedef struct aeFiredEvent {
        int fd;		// 有事件触发的文件描述符
        int mask;   // 这个fd触发具体的事件类型
    } aeFiredEvent;
    ```

+ `timeEventHead`

    主要是记录了定时器的头部节点，即第一个触发的定时器事件。

    ```cpp
    typedef struct aeTimeEvent {
        long long id;  /* time event identifier. */
        long when_sec; /* seconds */
        long when_ms;  /* milliseconds */
        aeTimeProc* timeProc;  // 时间回调函数
        aeEventFinalizerProc* finalizerProc;
        void* clientData;
        struct aeTimeEvent* prev;
        struct aeTimeEvent* next;
        int refcount; /* refcount to prevent timer events from being freed in recursive time event calls. */
    } aeTimeEvent;
    ```

### aeCreateEventLoop

创建EventLoop对象，完成的是：

+ 给`events,fired`分配内存，每个`event[i]`设置为不关注任何事件
+ 创建`epollfd`
+ 其他默认初始化

```cpp
  /// 创建 eventLoop
  aeEventLoop *aeCreateEventLoop(int setsize) {
      aeEventLoop *eventLoop;
      int i;

      if ((eventLoop = zmalloc(sizeof(*eventLoop))) == NULL) 
          goto err;
      eventLoop->events = zmalloc(sizeof(aeFileEvent)*setsize);
      eventLoop->fired = zmalloc(sizeof(aeFiredEvent)*setsize);
      if (eventLoop->events == NULL || eventLoop->fired == NULL) 
          goto err;
      
      eventLoop->setsize = setsize;
      eventLoop->lastTime = time(NULL);
      eventLoop->timeEventHead = NULL;
      eventLoop->timeEventNextId = 0;
      eventLoop->stop = 0;
      eventLoop->maxfd = -1;
      eventLoop->beforesleep = NULL;
      eventLoop->aftersleep = NULL;
      eventLoop->flags = 0;
      // 创建 epollfd
      if (aeApiCreate(eventLoop) == -1) goto err;
      /* Events with mask == AE_NONE are not set. So let's initialize the vector with it. */
      // 初始化时，什么事件也没关注
      for (i = 0; i < setsize; i++)
          eventLoop->events[i].mask = AE_NONE;
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
## epollfd 

针对`epollfd`有几个相关操作，有关的函数都是有 **`aeApi_xxx`** 前缀。

+ `aeApiCreate`：创建 `epollfd`
+ `aeApiAddEvent`：注册感兴趣的事件
+ `aeApiDelEvent`：删除感兴趣的事件
+ `aeApiPoll`：进入`epoll_wait`中阻塞等待

### aeApiCreate

`aeApiCreate`函数，基于`epoll_create`函数创建`epollfd`，其中`epoll_create`中入口参数`size`没有任何含义，但必须是个大于0的正数，也是可以使用`epoll_create1`这个函数来创建，不过要传入一个标志位。

`epollfd`相关状态在REdis中使用`aeApiState`结构体保存

```cpp
  typedef struct aeApiState {
      int epfd;				// epollfd
      struct epoll_event* events;
  } aeApiState;

   struct epoll_event {
     uint32_t     events;   // epoll_wait中触发的事件
     epoll_data_t data;	    // 存放触发事件对应的fd
  } __EPOLL_PACKED;
```
+ `epfd`：保存的此`EventLoop`中的`epollfd`
+ `events`：用于保存 `epoll_wait` 检测到的活跃事件类型及其对应的`fd`，其大小最大是`eventLoop`中的`setsize`、

`aeApiCreate`函数整个逻辑也很简单，使用的是`aeApiState`对象来保存`epollfd`及其事件信息。

```cpp
  static int aeApiCreate(aeEventLoop *eventLoop) {
      aeApiState* state = zmalloc(sizeof(aeApiState));
      if (!state) return -1;
   
      state->events = zmalloc(sizeof(struct epoll_event)*eventLoop->setsize);
      if (!state->events) {
          zfree(state);
          return -1;
      }
      
      state->epfd = epoll_create(1024); /* 1024 is just a hint for the kernel */
      if (state->epfd == -1) {
          zfree(state->events);
          zfree(state);
          return -1;
      }
      eventLoop->apidata = state; 		// 将 epollfd 存在这个地方
      return 0;
  }
```
### aeApiAddEvent
`aeApiAddEvent`函数， 将`fd `注册到`epollfd`上去。 `listenfd` 、 `clientfd` 都是使用这个函数来注册感兴趣事情，传入的参数`fd`就是需要注册的文件描述符。

```cpp
// 让 fd 关注事件 mask
static int aeApiAddEvent(aeEventLoop *eventLoop, int fd, int mask) {
    aeApiState *state = eventLoop->apidata;
    struct epoll_event ee = {0}; /* avoid valgrind warning */
    /* If the fd was already monitored for some event, we need a MOD
     * operation. Otherwise we need an ADD operation. */
    // 如果之前就关注了一些事件，则此次就是修改，否则就是添加
    int op = eventLoop->events[fd].mask == AE_NONE ? EPOLL_CTL_ADD : EPOLL_CTL_MOD;

    ee.events = 0;

    mask |= eventLoop->events[fd].mask;  /* Merge old events */ // 本次调用后，总共需要关注的事件
    if (mask & AE_READABLE) ee.events |= EPOLLIN;
    if (mask & AE_WRITABLE) ee.events |= EPOLLOUT;
    ee.data.fd = fd;
    if (epoll_ctl(state->epfd, op, fd, &ee) == -1) return -1;
    return 0;
}
```

### aeApiDelEvent

使文件描述符`fd`不再关注`delmask`事件。

```cpp
  static void aeApiDelEvent(aeEventLoop *eventLoop, int fd, int delmask) {
      aeApiState *state = eventLoop->apidata;
      struct epoll_event ee = {0}; /* avoid valgrind warning */
      int mask = eventLoop->events[fd].mask & (~delmask); // 取消对于 delmask 的关注

      ee.events = 0;
      if (mask & AE_READABLE) ee.events |= EPOLLIN;
      if (mask & AE_WRITABLE) ee.events |= EPOLLOUT;
      ee.data.fd = fd;
      // 如果取消关注事件 delmask 后还由其他事件，那么就修改  EPOLL_CTL_MOD
      // 否则就将fd从state->efd的事件空间中删除
      if (mask != AE_NONE) {
          epoll_ctl(state->epfd,EPOLL_CTL_MOD,fd,&ee);
      } else {
          epoll_ctl(state->epfd,EPOLL_CTL_DEL,fd,&ee);
      }
  }
```

### aeApiPoll
`aeApiPoll`函数基于`epoll_wait`实现，阻塞等待事件的发生

```cpp
static int aeApiPoll(aeEventLoop* eventLoop, struct timeval* tvp) {
    aeApiState *state = eventLoop->apidata;
    int retval, numevents = 0;

    retval = epoll_wait(state->epfd,                                         // epollfd
                        state->events,                                       // 存储触发的事件类型及其fd
                        eventLoop->setsize,                                  // 最大可触发大小
                        tvp ? (tvp->tv_sec*1000 + tvp->tv_usec/1000) : -1);  // 最长阻塞时间
    if (retval > 0) {
        int j;
         // 逐个记录事件
        numevents = retval;
        for (j = 0; j < numevents; j++) {
            int mask = 0;
            struct epoll_event* e = state->events+j;

            if (e->events & EPOLLIN) mask |= AE_READABLE;
            if (e->events & EPOLLOUT) mask |= AE_WRITABLE;
            if (e->events & EPOLLERR) mask |= AE_WRITABLE|AE_READABLE; // 错误是即可读又是可写
            if (e->events & EPOLLHUP) mask |= AE_WRITABLE|AE_READABLE; // 对端关闭
            eventLoop->fired[j].fd = e->data.fd; 					 // 记录触发事件的 fd
            eventLoop->fired[j].mask = mask;     					 // 记录触发的事件类型
        }
    }
    return numevents;
}
```

## anetListen

将监听文件描述符`s`绑定到地址 `sd`上，并开启监听。

```cpp
  static int anetListen(char *err, int s, struct sockaddr *sa, socklen_t len, int backlog) {
      if (bind(s,sa,len) == -1) {
          anetSetError(err, "bind: %s", strerror(errno));
          close(s);
          return ANET_ERR;
      }

      if (listen(s, backlog) == -1) {
          anetSetError(err, "listen: %s", strerror(errno));
          close(s);
          return ANET_ERR;
      }
      return ANET_OK;
  }
```
这两个函数没有什么特别之处，如果出错就直接返回关闭`socket`，返回-1，与`sockfd`是否非阻塞模式无关。这个函数只是创建`TcpServer`的一部分，其上层有一个更加全面的函数`_anetTcpServer`。

#### anetTcpServer

`anetTcpServer`函数，可以创建IPV4、IPV6服务器。

```cpp
int anetTcpServer(char *err, int port, char *bindaddr, int backlog)
{
    return _anetTcpServer(err, port, bindaddr, AF_INET, backlog);		// ipv4
}

int anetTcp6Server(char *err, int port, char *bindaddr, int backlog)
{
    return _anetTcpServer(err, port, bindaddr, AF_INET6, backlog);		// ipv6
}
```

#### _anetTcpServer

`_anetTcpServer`函数，创建监听状态的服务器，步骤如下：

+ `getaddrinfo`函数，获取本机上的所有`ip`地址及对应的`TCP`服务。此函数参数：

  + `SOCK_STREAM`：指定了服务类型，是属于tcp
  + `AI_PASSIVE`：如果传入的地址不是空字符串，那么这个设置无效果。否则，`getaddrinfo`返回的ip地址就是统配地址。

  这个函数，为创建的`listenfd`任选一个由`getaddrinfo`返回的本地`IP`地址来绑定。

+ 为了使得地址复用，设置了`SO_REUSEADDR`参数

+ 再调用了绑定和监听

`_anetTcpServer` 函数，为本地的IPV4、IPV6各自创建一个监听文件描述符并保存在 `server.ipfd`中，个数由`server.ipfd_count`记录。

```cpp
  static int _anetTcpServer(char *err, int port, char *bindaddr, int af, int backlog)
  {
      int s = -1, rv;
      char _port[6];  /* strlen("65535") */
      // 调用 getadddrinfo 函数的前提准备
      struct addrinfo hints, *servinfo, *p; 

      snprintf(_port,6,"%d",port);
      memset(&hints,0,sizeof(hints));
      hints.ai_family = af;
      hints.ai_socktype = SOCK_STREAM; // TCP 数据类型
      hints.ai_flags = AI_PASSIVE;     /* No effect if bindaddr != NULL，如果 bindarry==NULL，返回的就是通配地址 */

      if ((rv = getaddrinfo(bindaddr, _port, &hints, &servinfo)) != 0) {
          anetSetError(err, "%s", gai_strerror(rv));
          return ANET_ERR;
      }
      
      // 绑定成功一个即可
      for (p = servinfo; p != NULL; p = p->ai_next) {
          
          if ((s = socket(p->ai_family, p->ai_socktype, p->ai_protocol)) == -1) // 创建TCP类型的 listenfd
              continue;
          if (af == AF_INET6 && anetV6Only(err,s) == ANET_ERR) goto error;
          if (anetSetReuseAddr(err, s) == ANET_ERR) goto error;				  // 设置地址复用
          if (anetListen(err,s,p->ai_addr,p->ai_addrlen,backlog) == ANET_ERR) s = ANET_ERR;
          goto end;
      }
      if (p == NULL) {
          anetSetError(err, "unable to bind socket, errno: %d", errno);
          goto error;
      }

  error:
      if (s != -1) close(s);
      s = ANET_ERR;
  end:
      freeaddrinfo(servinfo);
      return s;
  }
```
至此，创建监听服务器已经完成，下一步应该是要创建`epollfd`，并且`listenfd`注册到`epollfd`中，并且关注可读事件。

## aeCreateFileEvent

这个函数的作用是将文件描述符`fd`挂在 `epollfd` 上，并且注册感兴趣的事件。在`initServer`函数中`aeCreateFileEvent`将`listenfd`挂在`epollfd`上并注册可读事件，目的是监听客户端的连接请求，客户端的连接请求处理函数是`acceptTcpHandler`

```cpp
	 //in server.c
	for (j = 0; j < server.ipfd_count; j++) {
      if (aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE, acceptTcpHandler,NULL) == AE_ERR)
          {
              serverPanic("Unrecoverable error creating server.ipfd file event.");
          }
    }
```
由上面分析可知 ` server.ipfd` 中保存的是根据本地的不同ip地址创建的 `listenfd`。`aeCreateFileEvent`函数作用是将每个监听`listenfd`绑定到所属的`loop`的`epollfd`中去。
```cpp
  int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask, aeFileProc *proc, void *clientData)
  {
      if (fd >= eventLoop->setsize) {
          errno = ERANGE;
          return AE_ERR;
      }
      aeFileEvent* fe = &eventLoop->events[fd];
      
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
这个函数在此处的主要作用：
+ 将 `listenfd` 注册到所属的 `eventLoop` 的 `epfd` 中去，并关注可读事件
+ 设置可读事件的回调函数 `acceptTcpHandler`，这个也是新的客户端连接到来时候的处理函数
+ 更新最大注册的 `fd`

至此，建立监听TCP服务器的流程基本完成：`aeCreateEventLoop --> listen --> aeCreateFileEvent`，

## aeMain

`aeMain`函数，即事件循环，不断的轮询处理各个请求并回应，核心是函数 `aeProcessEvents`。

```cpp
  void aeMain(aeEventLoop *eventLoop) {
      eventLoop->stop = 0;
      while (!eventLoop->stop) {
          aeProcessEvents(eventLoop, AE_ALL_EVENTS|
                                     AE_CALL_BEFORE_SLEEP|
                                     AE_CALL_AFTER_SLEEP);
      }
  }
```
### aeProcessEvents

`aeProcessEvents`是`eventloop`的核心，整个流程可以分为5个部分：

1. 在`aeApiPoll`之前就做了一件事：计算`epoll_wait`需要阻塞的时间
   + 如果设置了`AE_DONT_WAIT`，那么就是不阻塞，`epoll_wait`的超时时间为0
   + 如果没有其他任务，只有定时器任务，那么`epoll_wait`阻塞时间即最早超时时间，防止定时器任务等待过久
   + 如果也没有定时器任务，那么就永远等待，直到有事件到来。
2. `beforesleep`：主要是在`epoll_wait`阻塞前处理一些任务，防止因阻塞长时间无法执行，或者是一些准备工作。（后续介绍）
3. `epoll_wait`等待事件发生
4. `aftersleep`：可以用于完成 `epoll_wait`唤醒之后的一些校验工作
5. 处理事件：根据是否设置了`AE_BARRIER`，来决定同一个`fd`上是先处理可读事件还是先处理可写事件。

6. 处理完活跃的事件，最早的定时器可能已经超时了，那么就是可以去执行了。

```cpp
int aeProcessEvents(aeEventLoop *eventLoop, int flags)
{
    int processed = 0, numevents;

    /* Nothing to do? return ASAP */
    if (!(flags & AE_TIME_EVENTS) && !(flags & AE_FILE_EVENTS)) return 0;

    /* Note that we want call select() even if there are no
     * file events to process as long as we want to process time
     * events, in order to sleep until the next time event is ready
     * to fire.
     * 
     * 1 eventLoop->maxfd == -1，即没有事件fd，那么就没有等待处理的事件
     * 2 flags & AE_TIME_EVENTS) && !(flags & AE_DONT_WAIT)：如果存在定时器事件，
     *		要么等待定时器事件，
     *      要么在没有设置 AE_DONT_WAIT，则就一直阻塞在 aeApiPoll，直到有事件触发
     * */
    if (eventLoop->maxfd != -1 || ((flags & AE_TIME_EVENTS) && !(flags & AE_DONT_WAIT))) {
        int j;
        aeTimeEvent *shortest = NULL;
        struct timeval tv, *tvp;
        
        if (flags & AE_TIME_EVENTS && !(flags & AE_DONT_WAIT))
            shortest = aeSearchNearestTimer(eventLoop);    // 如果有时间事件存在，则获取最近的定时器超时时间
        if (shortest) {
            long now_sec, now_ms;

            aeGetTime(&now_sec, &now_ms);
            tvp = &tv;

            /* How many milliseconds we need to wait for the next
             * time event to fire? */
            // 最近超时的时间转为毫秒单位
            long long ms = (shortest->when_sec - now_sec)*1000 + shortest->when_ms - now_ms;

            if (ms > 0) {
                tvp->tv_sec = ms/1000;
                tvp->tv_usec = (ms % 1000)*1000;
            } else {
                tvp->tv_sec = 0;
                tvp->tv_usec = 0;
            }
        } else {
            // 要么是超时时间为0,即使设置了AE_DONT_WAIT,不要阻塞
            // 要么是没有定时时间,则可以阻塞
            /* If we have to check for events but need to return
             * ASAP because of AE_DONT_WAIT we need to set the timeout
             * to zero */
            // 不要阻塞,则设置超时时间为0
            if (flags & AE_DONT_WAIT) {
                tv.tv_sec = tv.tv_usec = 0;
                tvp = &tv;
            } else {
                /* Otherwise we can block */
                // 没有可触发事件,则等待
                tvp = NULL; /* wait forever */
            }
        }

        // 再次确认下
        if (eventLoop->flags & AE_DONT_WAIT) {
            tv.tv_sec = tv.tv_usec = 0;
            tvp = &tv;
        }

        if (eventLoop->beforesleep != NULL && flags & AE_CALL_BEFORE_SLEEP)
            eventLoop->beforesleep(eventLoop); // 在休眠之前执行

        /* Call the multiplexing API, will return only on timeout or when some event fires. */
        numevents = aeApiPoll(eventLoop, tvp);

        /* After sleep callback. */
        if (eventLoop->aftersleep != NULL && flags & AE_CALL_AFTER_SLEEP)
            eventLoop->aftersleep(eventLoop); // 在休眠之后执行

        // 逐个处理触发的事件
        for (j = 0; j < numevents; j++) {
            aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];
            int mask = eventLoop->fired[j].mask;
            int fd   = eventLoop->fired[j].fd;
            int fired = 0; /* Number of events fired for current fd. */

            /* Normally we execute the readable event first, and the writable
             * event laster. This is useful as sometimes we may be able
             * to serve the reply of a query immediately after processing the
             * query.
             *
             * However if AE_BARRIER is set in the mask, our application is
             * asking us to do the reverse: never fire the writable event
             * after the readable. In such a case, we invert the calls.
             * This is useful when, for instance, we want to do things
             * in the beforeSleep() hook, like fsynching a file to disk,
             * before replying to a client. */
            int invert = fe->mask & AE_BARRIER;

            /* Note the "fe->mask & mask & ..." code: maybe an already
             * processed event removed an element that fired and we still
             * didn't processed, so we check if the event is still valid.
             *
             * Fire the readable event if the call sequence is not inverted. */
            // fe->mask 是之前关注的事件，mask是产生的事件，如果都有 AE_READABLE
            // 则触发可读事件
            if (!invert && fe->mask & mask & AE_READABLE) {
                fe->rfileProc(eventLoop,fd,fe->clientData,mask);
                fired++;
                fe = &eventLoop->events[fd]; /* Refresh in case of resize. */
            }

            /* Fire the writable event. */
            if (fe->mask & mask & AE_WRITABLE) {
                if (!fired || fe->wfileProc != fe->rfileProc) {
                    fe->wfileProc(eventLoop,fd,fe->clientData,mask);
                    fired++;
                }
            }

            /* If we have to invert the call, fire the readable event now
             * after the writable one. */
            if (invert) {
                fe = &eventLoop->events[fd]; /* Refresh in case of resize. */
                if ((fe->mask & mask & AE_READABLE) &&
                    (!fired || fe->wfileProc != fe->rfileProc))
                {
                    fe->rfileProc(eventLoop,fd,fe->clientData,mask);
                    fired++;
                }
            }

            processed++;
        }
    } 

    // 可以去处理时间事件了
    /* Check time events */
    if (flags & AE_TIME_EVENTS)
        processed += processTimeEvents(eventLoop);

    return processed; /* return the number of processed file/time events */
}
```
### processTimeEvents

定时器事件中，定时器是基于链表实现的，

```cpp
/* Process time events */
static int processTimeEvents(aeEventLoop *eventLoop) {
    int processed = 0;
    aeTimeEvent *te;
    long long maxId;
    time_t now = time(NULL);

    /* If the system clock is moved to the future, and then set back to the
     * right value, time events may be delayed in a random way. Often this
     * means that scheduled operations will not be performed soon enough.
     *
     * Here we try to detect system clock skews, and force all the time
     * events to be processed ASAP when this happens: the idea is that
     * processing events earlier is less dangerous than delaying them
     * indefinitely, and practice suggests it is. */
    if (now < eventLoop->lastTime) {
        te = eventLoop->timeEventHead;
        while(te) {
            te->when_sec = 0;
            te = te->next;
        }
    }
    eventLoop->lastTime = now;

    te = eventLoop->timeEventHead;
    maxId = eventLoop->timeEventNextId-1;
    while(te) {
        long now_sec, now_ms;
        long long id;

        /* Remove events scheduled for deletion. */
        if (te->id == AE_DELETED_EVENT_ID) {
            aeTimeEvent *next = te->next;
            /* If a reference exists for this timer event,
             * don't free it. This is currently incremented
             * for recursive timerProc calls */
            if (te->refcount) {
                te = next;
                continue;
            }
            if (te->prev)
                te->prev->next = te->next;
            else
                eventLoop->timeEventHead = te->next;
            if (te->next)
                te->next->prev = te->prev;
            if (te->finalizerProc)
                te->finalizerProc(eventLoop, te->clientData);
            zfree(te);
            te = next;
            continue;
        }

        /* Make sure we don't process time events created by time events in
         * this iteration. Note that this check is currently useless: we always
         * add new timers on the head, however if we change the implementation
         * detail, this check may be useful again: we keep it here for future
         * defense. */
        if (te->id > maxId) {
            te = te->next;
            continue;
        }
        aeGetTime(&now_sec, &now_ms);
        if (now_sec > te->when_sec ||
            (now_sec == te->when_sec && now_ms >= te->when_ms))
        {
            int retval;

            id = te->id;
            te->refcount++;
            retval = te->timeProc(eventLoop, id, te->clientData);
            te->refcount--;
            processed++;
            if (retval != AE_NOMORE) {
                aeAddMillisecondsToNow(retval,&te->when_sec,&te->when_ms);
            } else {
                te->id = AE_DELETED_EVENT_ID;
            }
        }
        te = te->next;
    }
    return processed;
}
```

#### aeCreateTimeEvent

创建定时器事件


```cpp
if (aeCreateTimeEvent(server.el, 1, serverCron, NULL, NULL) == AE_ERR) {
    serverPanic("Can't create event loop timers.");
    exit(1);
}
```