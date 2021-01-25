# Networking

本小节介绍对客户端的封装以及服务器处理客户端连接请求的过程。

+ 客户端的封装
+ 服务器处理客户端的连接请求

### Connection

为了维护客户端的一些状态及其一些读写操作，需要为每个客户端封装一个连接对象`conn`，结构体如下。

```cpp
struct connection {
      ConnectionType* type;  // 操作 connection 中的函数指针
      ConnectionState state; // 是一个 enum，表示连接的状态
      short int flags;       // CONN_FLAG_CLOSE_SCHEDULED 或者 CONN_FLAG_WRITE_BARRIER
      short int refs;        //引用计数，控制着这个连接对象生命周期
      int last_errno;        // 最近一次的错误类型
      void *private_data;    // 保存的是这个连接对应的客户端 client
      ConnectionCallbackFunc conn_handler;  // 连接回调
      ConnectionCallbackFunc write_handler; // 写回调
      ConnectionCallbackFunc read_handler;  // 读回调
      int fd;               // cfd
  };
```

+ `state`：客户端的连接状态如下

  ```cpp
  typedef enum {
      CONN_STATE_NONE = 0,
      CONN_STATE_CONNECTING,  // connecting, 发起 connect 连接时
      CONN_STATE_ACCEPTING,   // accepting, 创建客户端时初始状态，即 accept 之前的状态
      CONN_STATE_CONNECTED,   // connected, 成功 accept 之后的状态
      CONN_STATE_CLOSED,      // closed,    关闭的状态
      CONN_STATE_ERROR        // error,     出错了
  } ConnectionState;
  ```

  也有相应获取状态`conn->state`的方法：

  ```cpp
  int connGetState(connection *conn) {
      return conn->state;
  }
  ```

+ `refs`：控制着连接对象`conn`的生命周期

  ```cpp
  // in conhelpers.h
  static inline void connIncrRefs(connection *conn) { conn->refs++; }
  
  static inline void connDecrRefs(connection *conn) { conn->refs--; }
  
  static inline int connHasRefs(connection *conn) { return conn->refs; }
  ```

+ `private_data`：保存着这个连接对象所属的客户端，在`createClient`中设置为客户端`c`。

  ```cpp
  /* Associate a private data pointer with the connection */
  void connSetPrivateData(connection *conn, void *data) {
      conn->private_data = data;
  }
  
  /* Get the associated private data pointer */
  void *connGetPrivateData(connection *conn) {
      return conn->private_data;
  }
  ```

+ 回调函数：`read_handler`、`write_handler`以及`read_handler`都是由函数指针集合`ConnectionType`来设置。

### ConnectionType

`ConnectionType`结构体封装了客户端连接对象的一些读写、Accept和关闭连接等操作，是函数指针的结构体。

```cpp
// 读写函数
typedef struct ConnectionType {
    // 读写
    void (*ae_handler)(struct aeEventLoop *el, int fd, void *clientData, int mask);
    // 处理连接请求
    int  (*connect) (struct connection *conn, 
                     const char *addr, int port, const char *source_addr, 
                     ConnectionCallbackFunc connect_handler);
    // 处理读写、关闭和Accept
    int  (*write)     (struct connection *conn, const void *data, size_t data_len);
    int  (*read)      (struct connection *conn, void *buf, size_t buf_len);
    void (*close)     (struct connection *conn);
    int  (*accept)    (struct connection *conn, ConnectionCallbackFunc accept_handler);
    // set
    int  (*set_write_handler)(struct connection *conn, ConnectionCallbackFunc handler, int barrier);
    int  (*set_read_handler)(struct connection *conn, ConnectionCallbackFunc handler);
    // get
    const char *(*get_last_error)(struct connection *conn);
    int  (*blocking_connect)(struct connection *conn, const char *addr, int port, long long timeout);
    // 异步读写
    ssize_t (*sync_write)(struct connection *conn, char *ptr, ssize_t size, long long timeout);
    ssize_t (*sync_read)(struct connection *conn, char *ptr, ssize_t size, long long timeout);
    ssize_t (*sync_readline)(struct connection *conn, char *ptr, ssize_t size, long long timeout);
} ConnectionType;
```

在`connection.c`中，定义了一个`ConnectionType`对象`CT_Socket`，及其初始化如下：

```cpp
ConnectionType CT_Socket = {
    .ae_handler         = connSocketEventHandler,
    .close              = connSocketClose,
    .write              = connSocketWrite,
    .read               = connSocketRead,
    .accept             = connSocketAccept,
    .connect            = connSocketConnect,
    .set_write_handler  = connSocketSetWriteHandler,
    .set_read_handler   = connSocketSetReadHandler,
    .get_last_error     = connSocketGetLastError,
    .blocking_connect   = connSocketBlockingConnect,
    .sync_write         = connSocketSyncWrite,
    .sync_read          = connSocketSyncRead,
    .sync_readline      = connSocketSyncReadLine
};
```

#### connSocketEventHandler

`connSocketEventHandler`函数，主要是综合处理可读、可写事件。这个函数中的 `callHandler`在后文详细介绍。

```cpp
// 处理epllfd上可读可写事件
static void connSocketEventHandler(struct aeEventLoop *el, int fd, void *clientData, int mask)
{
    UNUSED(el);
    UNUSED(fd);
    connection *conn = clientData;

    // 针对发起连接对象
    if (conn->state == CONN_STATE_CONNECTING &&
        (mask & AE_WRITABLE) && 
        conn->conn_handler) 
    {

        if (connGetSocketError(conn)) {
            conn->last_errno = errno;
            conn->state = CONN_STATE_ERROR;
        } else {
            conn->state = CONN_STATE_CONNECTED; 
        }

        // 如果没有设置可写事件的处理函数,则直接取消可写事件
        if (!conn->write_handler) aeDeleteFileEvent(server.el,conn->fd,AE_WRITABLE);
        // 处理新连接到来
        if (!callHandler(conn, conn->conn_handler)) return;
        conn->conn_handler = NULL; 
    }

    /* Normally we execute the readable event first, and the writable
     * event later. This is useful as sometimes we may be able
     * to serve the reply of a query immediately after processing the
     * query.
     * 
     * 通常我们先执行可读事件，然后执行可写事件。
     * 这种方式很有用，因为有时我们可以在处理查询之后立即提供查询的结果。

     * However if WRITE_BARRIER is set in the mask, our application is
     * asking us to do the reverse: never fire the writable event after the readable. 
     * In such a case, we invert the calls.
     * 
     * 然后,如果设置了 WRITE_BARRIER, 那么处理程序就反过来:在处理了可读事件之后都不触发可写事件
     * 
     * This is useful when, for instance, we want to do things
     * in the beforeSleep() hook, like fsync'ing a file to disk,
     * before replying to a client.
     * 
     * 这种操作很有用,比如,当我在beforeSleep()函数中,执行一些阻塞操作,类似fsync操作
     *  */
    int invert = conn->flags & CONN_FLAG_WRITE_BARRIER;

    int call_write = (mask & AE_WRITABLE) && conn->write_handler;
    int call_read  = (mask & AE_READABLE) && conn->read_handler;

    /* Handle normal I/O flows */
    if (!invert && call_read) {
        if (!callHandler(conn, conn->read_handler)) return;
    }
    /* Fire the writable event. */
    if (call_write) {
        if (!callHandler(conn, conn->write_handler)) return;
    }
    /* If we have to invert the call, fire the readable event now
     * after the writable one. */
    if (invert && call_read) {
        if (!callHandler(conn, conn->read_handler)) return;
    }
}
```

#### connSocketWrite

这是REdis中实际完成写操作的最底层的函数，调用`write`函数完成发送。

```cpp
static int connSocketWrite(connection *conn, const void *data, size_t data_len) {
    int ret = write(conn->fd, data, data_len);
    if (ret < 0 && errno != EAGAIN) {  // ret ==-1 && errno ==EAGAIN，在非阻塞IO是正常下的，不是错误
        conn->last_errno = errno;
        conn->state = CONN_STATE_ERROR;
    }

    return ret;
}
```

`connSocketWrite` 函数上层是被`connWrite`函数调用。用户无法直接调用 `connSocketWrite`，只能通过`conn`对象来调用。

```cpp
static inline int connWrite(connection *conn, const void *data, size_t data_len) {
    return conn->type->write(conn, data, data_len);
}
```

#### connSocketRead

和写操作类似，这是最底层的函数，服务器读取客户端发来的数据，

```cpp
// conn->type->write
static int connSocketWrite(connection *conn, const void *data, size_t data_len) {
    int ret = write(conn->fd, data, data_len);
    if (ret < 0 && errno != EAGAIN) {	// ret ==-1 && errno ==EAGAIN，在非阻塞IO是正常下的，不是错误
        conn->last_errno = errno;
        conn->state = CONN_STATE_ERROR;
    }

    return ret;
}
```

`connSocketWrite` 经过封装被 `connRead`调用，将数据读取到到`buf`中。用户不应该直接调用 `connSocketWrite` 。

```cpp
static inline int connRead(connection *conn, void *buf, size_t buf_len) {
    return conn->type->read(conn, buf, buf_len);
}
```

#### connSocketClose

关闭客户端操作。

注意：控制连接对象生命周期的引用计数`conn->refs`。如果此时引用计数不为0，说明是处于某个回调函数中，此时不能直接关闭，设置标志位 <font color=yellow> CONN_FLAG_CLOSE_SCHEDULED</font>，需要延迟关闭。这个在后面应用时会有更深的理解。

```cpp
static void connSocketClose(connection *conn) {
    if (conn->fd != -1) {
        aeDeleteFileEvent(server.el,conn->fd,AE_READABLE);  // 取消关注可写事件
        aeDeleteFileEvent(server.el,conn->fd,AE_WRITABLE);  // 取消关注可读事件
        close(conn->fd);                                    // 再关闭socket
        conn->fd = -1;
    }

    /* If called from within a handler, schedule the close but
     * keep the connection until the handler returns.
     */
    // 此处，即通过引用计数来控制conn的生命周期
    // 此时，尽管已经要关闭conn，但是其引用计数不是0
    // 说明处于某个回调函数中，等该回调函数返回之后，这个客户端就可以关闭
    if (connHasRefs(conn)) {
        conn->flags |= CONN_FLAG_CLOSE_SCHEDULED;
        return;
    }

    zfree(conn);
}
```

其上层也是有一个`wrapper`函数 `connClose`，提供给用户使用。

```cpp
static inline void connClose(connection *conn) {
    conn->type->close(conn);
}
```

#### connSocketSetWriteHandler

`connSocketSetWriteHandler`函数，主要是在`epoll_wait`上注册可写事件，并设置可写事件的回调函数为 `ae_handler`，真正执行写操作的还是`func`：

+ 设置连接对象`conn`的写回调函数 `conn->write_handler` 为 `func`，再注册写事件
+ 如果设置的回调函数 `func`为`NULL`，则取消注册可写事件

注意：这里注册的可写事件的回调函数是`ae_handler`，是因为在`ae_handler`中，综合处理了可读、可写事件。

```cpp
static int connSocketSetWriteHandler(connection *conn, ConnectionCallbackFunc func, int barrier) {
    if (func == conn->write_handler) return C_OK; 

    conn->write_handler = func;         // 设置新的可写事件处理函数，
    if (barrier)                
        conn->flags |= CONN_FLAG_WRITE_BARRIER;
    else
        conn->flags &= ~CONN_FLAG_WRITE_BARRIER;

    // 如果没有设置可写事件处理函数,则取消关注可写事件
    if (!conn->write_handler)
        aeDeleteFileEvent(server.el, conn->fd, AE_WRITABLE);
    else
    {
        // 关注可写事件
        if (aeCreateFileEvent(server.el,                // eventLoop
                              conn->fd,                 // epollfd
                              AE_WRITABLE,              // 关注可写事件
                              conn->type->ae_handler,   // 事件处理函数
                              conn) == AE_ERR) return C_ERR;
    }

    return C_OK;
}
```

根据函数 `connSocketSetWriteHandler` 的第三个标志位`barrier`，可封装成两个函数给用户使用，

```cpp
static inline int connSetWriteHandler(connection *conn, ConnectionCallbackFunc func) {
    return conn->type->set_write_handler(conn, func, 0);
}

static inline int connSetWriteHandlerWithBarrier(connection *conn, ConnectionCallbackFunc func, int barrier) {
    return conn->type->set_write_handler(conn, func, barrier);
}
```

#### connSocketSetReadHandler

`connSocketSetReadHandler`主要是注册可读事件，并设置读回调函数为 `ae_handler`，真正执行读取操作的还是`func`。

```cpp
// 关注可读事件并设置可读取事件处理函数
static int connSocketSetReadHandler(connection *conn, ConnectionCallbackFunc func) {
    if (func == conn->read_handler) return C_OK;

    conn->read_handler = func;
    if (!conn->read_handler)
        aeDeleteFileEvent(server.el,conn->fd,AE_READABLE);
    else
        if (aeCreateFileEvent(server.el,
                              conn->fd,
                              AE_READABLE,
                              conn->type->ae_handler,conn) == AE_ERR) return C_ERR;
    return C_OK;
}
```

上层是通过`conn`对象调用

```cpp
static inline int connSetReadHandler(connection *conn, ConnectionCallbackFunc func) {
    return conn->type->set_read_handler(conn, func);
}
```

至此，一个连接对象`conn`基本介绍完毕，其余的等待应用到介绍。

## Accept

下面介绍服务端处理客户端连接请求的流程。

### acceptTcpHandler

`acceptTcpHandler`函数，是服务器处理客户端的连接请求的回调函数，即监听文件描述符`server.ipfd[j]`的处理可读事件的回调函数。这个函数在`server.c`函数中设置。

```cpp
// in server.c
/* Create an event handler for accepting new connections in TCP and Unix domain sockets. */
for (j = 0; j < server.ipfd_count; j++) {
    if (aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE, acceptTcpHandler, NULL) == AE_ERR)
        serverPanic("Unrecoverable error creating server.ipfd file event.");
}
```

在 `acceptTcpHandler`函数中，获取客户端`cfd`，并建立连接。

```cpp
#define MAX_ACCEPTS_PER_CALL 1000

void acceptTcpHandler(aeEventLoop *el, int fd, void *privdata, int mask) {
    int cport, cfd, max = MAX_ACCEPTS_PER_CALL;	 // cport 客户端的端口，cfd 客户端的 fd
    char cip[NET_IP_STR_LEN]; 					// 客户端的 ip
    UNUSED(el);
    UNUSED(mask);
    UNUSED(privdata);
    
    // 为了防止短时间内过多的客户端连接请求，造成阻塞
    // 每次事件循环只能处理 MAX_ACCEPTS_PER_CALL 个客户端连接请求
    while(max--) {
        cfd = anetTcpAccept(server.neterr, fd, cip, sizeof(cip), &cport);
        if (cfd == ANET_ERR) {
            if (errno != EWOULDBLOCK)
                serverLog(LL_WARNING, "Accepting client connection: %s", server.neterr);
            return;
        }
        serverLog(LL_VERBOSE,"Accepted %s:%d", cip, cport);
        // 处理客户端
        acceptCommonHandler(connCreateAcceptedSocket(cfd), 0, cip);
    }
  }
```

### connCreateAcceptedSocket

主要是创建客户端连接对象`conn`，此外

+ 设置`conn->fd`为客户端`cfd`
+ 将`conn`对象的状态初始化为 `CONN_STATE_ACCEPTING`

```cpp
connection *connCreateSocket() {
    connection *conn = zcalloc(sizeof(connection));
    conn->type = &CT_Socket;
    conn->fd = -1;

    return conn;
}

connection* connCreateAcceptedSocket(int fd) {
    connection *conn = connCreateSocket();
    conn->fd = fd;							// accept 得到的 fd
    conn->state = CONN_STATE_ACCEPTING;  	  // 此时状态
    return conn;
}
```

### acceptCommonHandler

这个函数主要查看`accept`所得的客户端是否合理，满足各个条件，最终创建客户端。

```cpp
// 创建客户端
static void acceptCommonHandler(connection *conn, int flags, char *ip) {
    client *c;
    UNUSED(ip);

    /* Admission control will happen before a client is created and connAccept() called, 
     * because we don't want to even start transport-level negotiation if rejected.
     */
    // 如果客户端连接请求超过限制，则直接关闭这个客户端即可，
    if (listLength(server.clients) >= server.maxclients) {
        char *err = "-ERR max number of clients reached\r\n";

        /* That's a best effort error message, don't check write errors.
         * Note that for TLS connections, no handshake was done yet so nothing is written
         * and the connection will just drop.
         */
        // 发送给客户端错误信息
        if (connWrite(conn,err,strlen(err)) == -1) {
            /* Nothing to do, Just to avoid the warning... */
        }
        server.stat_rejected_conn++;    // 用于调试信息
        connClose(conn);			   // 关闭客户端
        return;
    }

    /* Create connection and client */
    // 创建客户端
    if ((c = createClient(conn)) == NULL) {
        // 创建失败，内存不足，则应该直接关闭这次行为
        char conninfo[100];
        serverLog(LL_WARNING,
                 "Error registering fd event for the new client: %s (conn: %s)",
                  connGetLastError(conn),
                  connGetInfo(conn, conninfo, sizeof(conninfo)));
        connClose(conn); /* May be already closed, just ignore errors */
        return;
    }

    /* Last chance to keep flags */
    c->flags |= flags;

    /* Initiate accept.
     *
     * Note that connAccept() is free to do two things here:
     * 1. Call clientAcceptHandler() immediately;
     * 2. Schedule a future call to clientAcceptHandler().
     *
     * Because of that, we must do nothing else afterwards.
     */
    // connAccept 主要是调用函数 clientAcceptHandler 对获得的客户端状态进行判断
    if (connAccept(conn, clientAcceptHandler) == C_ERR) {
        char conninfo[100];
        if (connGetState(conn) == CONN_STATE_ERROR) { 
            serverLog(LL_WARNING,
                      "Error accepting a client connection: %s (conn: %s)",
                       connGetLastError(conn), 
                       connGetInfo(conn, conninfo, sizeof(conninfo)));
        }
        freeClient(connGetPrivateData(conn)); // 同步关闭
        return;
    }
}
```

#### createClient

创建客户端的主要任务如下：

+ 将客户端`cfd`设置为非阻塞`IO`模式
+ 设置客户端的`TCP`参数，比如`cfd`上开启`nagle`算法
+ 开启保活定时器，即心跳检测，但是改变了默认值
+ 注册可读事件，并设置了读回调函数`readQueryFromClient`。这点符合`LT`模式的逻辑，没有注册可写事件，防止`busy loop`
+ `client`对象其他参数初始化

```cpp
client *createClient(connection *conn) {
    client *c = zmalloc(sizeof(client));

    /* passing NULL as conn it is possible to create a non connected client.
     * This is useful since all the commands needs to be executed
     * in the context of a client. When commands are executed in other
     * contexts (for instance a Lua script) we need a non connected client. */
    // Lua 脚本环境下，conn是NULL
    if (conn) {
        connNonBlock(conn);         // 设置客户端为非阻塞
        connEnableTcpNoDelay(conn); // 开启 nagle
        if (server.tcpkeepalive)    // 心跳检测
            connKeepAlive(conn,server.tcpkeepalive);
        connSetReadHandler(conn, readQueryFromClient); // 注册可读事件，可读事件的回调函数是 readQueryFromClient 
        connSetPrivateData(conn, c);                   // conn->private_data = data;
    }
 // --- 下面是一些参数初始化
```

##### connEnableTcpNoDelay

```cpp
int connEnableTcpNoDelay(connection *conn) {
    if (conn->fd == -1) return C_ERR;
    return anetEnableTcpNoDelay(NULL, conn->fd);
}

int anetEnableTcpNoDelay(char *err, int fd)
{
    return anetSetTcpNoDelay(err, fd, 1);
}
```

##### connNonBlock

将客户端设置为非阻塞IO模式，经过层层套娃，最终是调用 `anetSetBlock` 函数实现。

```cpp
int connNonBlock(connection *conn) {
    if (conn->fd == -1) return C_ERR;
    return anetNonBlock(NULL, conn->fd);
}

int anetNonBlock(char *err, int fd) {
    return anetSetBlock(err,fd,1);
}
```

anetSetBlock

将文件描述符fd设置为非阻塞IO模式

```cpp
int anetSetBlock(char *err, int fd, int non_block) {
    int flags;

    /* Set the socket blocking (if non_block is zero) or non-blocking.
     * Note that fcntl(2) for F_GETFL and F_SETFL can't be
     * interrupted by a signal. */
    if ((flags = fcntl(fd, F_GETFL)) == -1) {
        anetSetError(err, "fcntl(F_GETFL): %s", strerror(errno));
        return ANET_ERR;
    }

    if (non_block)
        flags |= O_NONBLOCK;
    else
        flags &= ~O_NONBLOCK;

    if (fcntl(fd, F_SETFL, flags) == -1) {
        anetSetError(err, "fcntl(F_SETFL,O_NONBLOCK): %s", strerror(errno));
        return ANET_ERR;
    }
    return ANET_OK;
}
```

anetSetTcpNoDelay

```cpp
static int anetSetTcpNoDelay(char *err, int fd, int val)
{
    if (setsockopt(fd, IPPROTO_TCP, TCP_NODELAY, &val, sizeof(val)) == -1)
    {
        anetSetError(err, "setsockopt TCP_NODELAY: %s", strerror(errno));
        return ANET_ERR;
    }
    return ANET_OK;
}
```

##### connKeepAlive

在 `connKeepAlive`底层调用的是 `anetKeepAlive`实现。

```cpp
int connKeepAlive(connection *conn, int interval) {
    if (conn->fd == -1) return C_ERR;
    return anetKeepAlive(NULL, conn->fd, interval);
}
```

anetKeepAlive

下面的代码中的心跳检测部分值得学习：

+ 是为每个客户端设置心跳检测
+ 通过三个选项，改变了`keeplive`的默认值

```cpp
  /* Set TCP keep alive option to detect dead peers. The interval option
  * is only used for Linux as we are using Linux-specific APIs to set
  * the probe send time, interval, and count. */
  int anetKeepAlive(char *err, int fd, int interval)
  {
    int val = 1;

    if (setsockopt(fd, SOL_SOCKET, SO_KEEPALIVE, &val, sizeof(val)) == -1)
    {
        anetSetError(err, "setsockopt SO_KEEPALIVE: %s", strerror(errno));
        return ANET_ERR;
    }

  #ifdef __linux__
    /* Default settings are more or less garbage, with the keepalive time
      * set to 7200 by default on Linux. Modify settings to make the feature
      * actually useful. */

    /* Send first probe after interval. */
    val = interval;
    if (setsockopt(fd, IPPROTO_TCP, TCP_KEEPIDLE, &val, sizeof(val)) < 0) {
        anetSetError(err, "setsockopt TCP_KEEPIDLE: %s\n", strerror(errno));
        return ANET_ERR;
    }

    /* Send next probes after the specified interval. Note that we set the
      * delay as interval / 3, as we send three probes before detecting
      * an error (see the next setsockopt call). */
    val = interval/3;
    if (val == 0) val = 1;
    if (setsockopt(fd, IPPROTO_TCP, TCP_KEEPINTVL, &val, sizeof(val)) < 0) {
        anetSetError(err, "setsockopt TCP_KEEPINTVL: %s\n", strerror(errno));
        return ANET_ERR;
    }

    /* Consider the socket in error state after three we send three ACK
      * probes without getting a reply. */
    val = 3;
    if (setsockopt(fd, IPPROTO_TCP, TCP_KEEPCNT, &val, sizeof(val)) < 0) {
        anetSetError(err, "setsockopt TCP_KEEPCNT: %s\n", strerror(errno));
        return ANET_ERR;
    }
  #else
    ((void) interval); /* Avoid unused var warning for non Linux systems. */
  #endif

    return ANET_OK;
  }
```

#### connAccept

connAccept内部调用如下：

```cpp
static inline int connAccept(connection *conn, ConnectionCallbackFunc accept_handler) {
    return conn->type->accept(conn, accept_handler);
}
```

但是，实际上调用的回调函数是 `connSocketAccept`，主要完成两个任务：

+ 将`conn`的状态从`CONN_STATE_ACCEPTING`转变为`CONN_STATE_CONNECTED`
+ 在`callHandler`中调用`accept_handler`函数，此处即 `clientAcceptHandler`，校验`conn`状态

```cpp
// conn->type->accept
static int connSocketAccept(connection *conn, ConnectionCallbackFunc accept_handler) {
    int ret = C_OK;

    if (conn->state != CONN_STATE_ACCEPTING)    // 判断连接状态
        return C_ERR;   
    conn->state = CONN_STATE_CONNECTED;         // 转换为：连接完成状态

    connIncrRefs(conn);
    if (!callHandler(conn, accept_handler))     // 调用回调函数 accept_handler
        ret = C_ERR;
    connDecrRefs(conn);

    return ret;
}
```

#### clientAcceptHandler

这个回调函数，主要是用于判断调用`connAccept`是否顺利，此时的状态`conn->state`应该是 `CONN_STATE_CONNECTED`，如果不是则需要关闭客户端、释放相关资源。因此，`clientAcceptHandler`相当于一个善后校验处理函数。

```cpp
// 对accept后建立的连接状态进行判断
void clientAcceptHandler(connection *conn) {
    client *c = connGetPrivateData(conn);

    if (connGetState(conn) != CONN_STATE_CONNECTED) {
        serverLog(LL_WARNING,
                  "Error accepting a client connection: %s",
                   connGetLastError(conn));
        freeClientAsync(c);  // 异步的
        return;
    }

    /* If the server is running in protected mode (the default) and there is no password set, 
     * nor a specific interface is bound, we don't accept
     * requests from non loopback interfaces. Instead we try to explain the
     * user what to do to fix it if needed. */
    if (server.protected_mode &&
        server.bindaddr_count == 0 &&
        DefaultUser->flags & USER_FLAG_NOPASS &&
        !(c->flags & CLIENT_UNIX_SOCKET))
    {
        char cip[NET_IP_STR_LEN+1] = { 0 };
        connPeerToString(conn, cip, sizeof(cip)-1, NULL);
       
        // 如果 cip 同时满足 127.0.0.1  与 ::1 
        // 下面 if 中的不可能执行
        if (strcmp(cip,"127.0.0.1") && strcmp(cip,"::1")) {
            char *err = "-DENIED Redis is running in protected mode because protected "
                        "mode is enabled, no bind address was specified, no "
                        "authentication password is requested to clients. In this mode "
                        "connections are only accepted from the loopback interface. "
                        "If you want to connect from external computers to Redis you "
                        "may adopt one of the following solutions: "
                        "1) Just disable protected mode sending the command "
                        "'CONFIG SET protected-mode no' from the loopback interface "
                        "by connecting to Redis from the same host the server is "
                        "running, however MAKE SURE Redis is not publicly accessible "
                        "from internet if you do so. Use CONFIG REWRITE to make this "
                        "change permanent. "
                        "2) Alternatively you can just disable the protected mode by "
                        "editing the Redis configuration file, and setting the protected "
                        "mode option to 'no', and then restarting the server. "
                        "3) If you started the server manually just for testing, restart "
                        "it with the '--protected-mode no' option. "
                        "4) Setup a bind address or an authentication password. "
                        "NOTE: You only need to do one of the above things in order for "
                        "the server to start accepting connections from the outside.\r\n";
            if (connWrite(c->conn,err,strlen(err)) == -1) {
                /* Nothing to do, Just to avoid the warning... */
            }
            server.stat_rejected_conn++;
            freeClientAsync(c);
            return;
        }
    }

    server.stat_numconnections++;   // 仅调试信息
    moduleFireServerEvent(REDISMODULE_EVENT_CLIENT_CHANGE,
                          REDISMODULE_SUBEVENT_CLIENT_CHANGE_CONNECTED,
                          c);
}
```

#### callHandler

只是一个辅助函数，用于调用回调函数 `handler`。

这里说下，`conn`的生命周期，因为这里是依靠引用计数维持`conn`的生命周期，因此每次在将`conn`作为参数时，都需要调用一次 `connIncrRefs(conn);`来增加引用，防止在 回调函数 `handler`中 `conn`被释放。比如 `handler`中又包含了一个 `callHandler`，那么没有这个增加引用计数，则潜藏着在第二个 `callHandler`中 `conn`就会关闭，导致返回到第一个 `callHandler`中时`conn`就失效了。

注意：整个代码中，只有两处有判断条件  `if (!connHasRefs(conn))` ：函数 `connClose` 和  `callHandler`，最后也肯定是在 `callHandler` 下面`if` 中关闭的。

```cpp
static inline int callHandler(connection *conn, ConnectionCallbackFunc handler) {
    connIncrRefs(conn);				
    if (handler) handler(conn); 				 // 如果存在处理程序handler,则处理
    connDecrRefs(conn);
    if (conn->flags & CONN_FLAG_CLOSE_SCHEDULED) { 
        if (!connHasRefs(conn)) connClose(conn);    // 如果客户端conn没有引用了,则直接关闭客户端
        return 0;
    }
    return 1;
}
```

至此，服务端处理完了客户端连接请求，主要过程如下

+ 先是为这个客户`cfd`创建一个连接对象`conn`，保存了客户端文件描述符`conn->fd`以及当前连接状态`conn->state`。
+ 为这个连接创建一个客户端对象`c`，还要为这个连接注册可读事件，设置读取回调事件为`readQueryFromClient`。毕竟客户端需要处理很多事情，并且将这个客户端对象保存在`conn->private_data`。此时，`conn`的状态是 `CONN_STATE_ACCEPTING`
+ 对这个客户端其他部分进行初始化，
+ 上面都完成了，那么就是创建完成，状态就变成`CONN_STATE_ACCEPTING`转为`CONN_STATE_CONNECTED`

