# OutputBuffer

本小节分析服务端的发送缓存区设计

## 添加数据

在 `client` 中和发送有关的字段:

```cpp
#define PROTO_REPLY_CHUNK_BYTES (16*1024) /* 16k output buffer */ 

typedef struct client {
    // ...
    list* reply;           				// 动态 output buffer
    unsigned long long reply_bytes;  	 // reply 中所有节点的字节数
    size_t sentlen;        				// 当前缓冲区中已经发送的字节数
	//...
	
    int bufpos;				       		// buf 中的字节数
    char buf[PROTO_REPLY_CHUNK_BYTES]; 	  // 静态缓冲区 static buffer
};
```

一般存储数据，都是先使用静态缓冲区`buf`，大小是16K，`buf`不足时再使用动态缓冲区`reply`。

+ `reply`：动态发送缓冲区，主要是针对当要发送的数据较多，静态缓冲区`buf`内存不足时就会使用`reply`。`reply`链表的节点类型是`clientReplyBlock`：

  ```cpp
  typedef struct clientReplyBlock {
      size_t size;	// 这个节点的容量
      size_t used;	// 这个节点已使用的容量
      char buf[];     // 用于存储数据
  } clientReplyBlock;
  ```


#### _addReplyToBuffer

这个函数向静态缓冲区 `buf`  中添加数据，能顺利存放在`buf`中，则返回C_OK，否则返回C_ERR.。

```cpp
int _addReplyToBuffer(client *c, const char *s, size_t len) {
    size_t available = sizeof(c->buf)-c->bufpos;  // 可用空间
    
	// CLIENT_CLOSE_AFTER_REPLY 表示将当前客户端 output buffer中数据发送完毕，就关闭，
    // 因此不会再接受新的数据
    if (c->flags & CLIENT_CLOSE_AFTER_REPLY) return C_OK;

    /* If there already are entries in the reply list, we cannot
     * add anything more to the static buffer. */
 	//  c->reply 存在数据，说明 buf 已经满了
    if (listLength(c->reply) > 0) return C_ERR;

    /* Check that the buffer has enough space available for this string. */
    // 查看 buf 是否能容纳当前待存放的数据
    if (len > available) return C_ERR;
	
    memcpy(c->buf+c->bufpos,s,len); // 复制到buf中
    c->bufpos +=len;			   // 改变 bufpos
    return C_OK;
}
```

#### _addReplyProtoToList

`_addReplyProtoToList`函数将在 **`c->reply`** 的尾部创建节点，存储待发送的数据。

```cpp
// 添加数据到动态内存c->reply的尾部节点
void _addReplyProtoToList(client *c, const char *s, size_t len) {
    if (c->flags & CLIENT_CLOSE_AFTER_REPLY) return;

    listNode *ln = listLast(c->reply);
    clientReplyBlock *tail = ln? listNodeValue(ln): NULL;

    /* Note that 'tail' may be NULL even if we have a tail node, becuase when
     * addReplyDeferredLen() is used, it sets a dummy node to NULL just
     * fo fill it later, when the size of the bulk length is set.
     *
     * tail 可能是 NULL，因为在 addReplyDeferredLen() 函数中你会创建一个 dummy node，其值是NULL
     * 因此，即使存在尾部节点，其值也可能是 NULL     
     */

    /* Append to tail string when possible. */
    // 如果不是NULL， 则尝试直接在尾部存储
    if (tail) {
        /* Copy the part we can fit into the tail, and leave the rest for new node */
        size_t avail = tail->size - tail->used;
        size_t copy = avail >= len? len: avail;  // 取最小
        memcpy(tail->buf + tail->used, s, copy);
        tail->used += copy;
        s += copy;
        len -= copy;
    }
    
    /* 注意：在使用了 addReplyDeferredLen() 函数后，tail 是 NULL，因此上面的 if(tail) 不会执行，
     * 		而是直接在下面的 if(len) 分支中创建新的节点接在 tail 后面，而 tail 依然是个 dummy node
     */

    // 如果 tail是 NULL 或者无法完全存储数据，就需要创建一个新的节点
    if (len) {
        /* Create a new node, make sure it is allocated to at least PROTO_REPLY_CHUNK_BYTES */
        size_t size = len < PROTO_REPLY_CHUNK_BYTES ? PROTO_REPLY_CHUNK_BYTES : len; //取最大
        tail = zmalloc(size + sizeof(clientReplyBlock));        // 创建新的节点
        
        /* take over the allocation's internal fragmentation */
        tail->size = zmalloc_usable(tail) - sizeof(clientReplyBlock);
        tail->used = len;
        memcpy(tail->buf, s, len);
        listAddNodeTail(c->reply, tail);					// 新节点添加在 tail 尾
        c->reply_bytes += tail->size;						// 更新总的已经分配字节数
    }

    // 查看 c->reply 是否超出内存限制，如果是，则异步关闭这个客户端
    asyncCloseClientOnOutputBufferLimitReached(c);
}
```

`asyncCloseClientOnOutputBufferLimitReached` 函数是个流量控制，如果对端接受不及时，能快速关闭客户端。

#### asyncCloseClientOnOutputBufferLimitReached

先检测是否达到内存限制，如果是则异步关闭客户端。

```cpp
void asyncCloseClientOnOutputBufferLimitReached(client *c) {
    if (!c->conn) return;   /* It is unsafe to free fake clients. */

    serverAssert(c->reply_bytes < SIZE_MAX-(1024*64)); // SIZE_MAX - 1024*64，是因为还有一个 buf，其大小是16k
    if (c->reply_bytes == 0 || c->flags & CLIENT_CLOSE_ASAP) return; 
    // 如果超过了限制，则关闭
    if (checkClientOutputBufferLimits(c)) {
        sds client = catClientInfoString(sdsempty(), c);
        // 异步清除客户端
        freeClientAsync(c);
        serverLog(LL_WARNING,
                  "Client %s scheduled to be closed ASAP for overcoming of output buffer limits.",
                  client);
        sdsfree(client);
    }
}
```

调用 `freeClientAsync` 异步关闭客户端，是因为客户端的资源较多，同步关闭可能会阻塞主线程。

#### checkClientOutputBufferLimits

检测 `c->reply`是否超过硬限制(hard limit)或软限制(soft limit)。是则返回1，否则返回0.

```cpp
int checkClientOutputBufferLimits(client *c) {
    int soft = 0, hard = 0, class;
    unsigned long used_mem = getClientOutputBufferMemoryUsage(c);  // 见下面分析

    class = getClientType(c);
    /* For the purpose of output buffer limiting, masters are handled
     * like normal clients. */
    if (class == CLIENT_TYPE_MASTER) class = CLIENT_TYPE_NORMAL;
    // 硬限制
    if (server.client_obuf_limits[class].hard_limit_bytes &&
        used_mem >= server.client_obuf_limits[class].hard_limit_bytes)
        hard = 1;
    // 软限制
    if (server.client_obuf_limits[class].soft_limit_bytes &&
        used_mem >= server.client_obuf_limits[class].soft_limit_bytes)
        soft = 1;

    /* We need to check if the soft limit is reached continuously for the
     * specified amount of seconds. */
    // 对于这个软限制，进行综合考虑，给予客户端一次机会，防止直接就关闭客户端
    if (soft) {
        if (c->obuf_soft_limit_reached_time == 0) {
            // 如果是首次达到 soft limit，则记录首次达到的时间，然后忽略这次
            c->obuf_soft_limit_reached_time = server.unixtime;
            soft = 0;  /* First time we see the soft limit reached */
        } else {
            time_t elapsed = server.unixtime - c->obuf_soft_limit_reached_time;
            // 如果第二次达到，并且两次之间的时间差小于阈值 soft_limit_seconds
            // 则认为是第一次 soft limit
            if (elapsed <= server.client_obuf_limits[class].soft_limit_seconds) {
                soft = 0; /* The client still did not reached the max number of
                             seconds for the soft limit to be considered
                             reached. */
            }
        }
    } else {
        c->obuf_soft_limit_reached_time = 0;
    }

    return soft || hard;
}
```

#### getClientOutputBufferMemoryUsage

计算 `c->reply`这个链表占用总字节大小，即存储数据的缓冲区大小+数据结构大小。

```cpp
unsigned long getClientOutputBufferMemoryUsage(client *c) {
    unsigned long list_item_size = sizeof(listNode) + sizeof(clientReplyBlock);
    // c->rely_bytes 是内存大小，
    // (list_item_size*listLength(c->reply)); 是数据结构大小
    return c->reply_bytes + (list_item_size*listLength(c->reply));
}
```

### addReply

`addReply`函数，将`server`发送给`client`的回应添加到发送缓冲区中：先使用静态缓冲区，再使用动态缓冲区。

```cpp
void addReply(client *c, robj *obj) {
    if (prepareClientToWrite(c) != C_OK)  // 是否有数据回应 client
        return; 
	
    if (sdsEncodedObject(obj)) {
        // 是字符串编码，则直接添加到缓冲区中
        if (_addReplyToBuffer(c,obj->ptr,sdslen(obj->ptr)) != C_OK) // 先使用 buf
            _addReplyProtoToList(c,obj->ptr,sdslen(obj->ptr));		// 再使用 c->reply
    } else if (obj->encoding == OBJ_ENCODING_INT) {
        /* For integer encoded strings we just convert it into a string
         * using our optimized function, and attach the resulting string
         * to the output buffer. */
        // 是整数编码，则先转换为字符串，再添加到缓冲区中
        char buf[32];
        size_t len = ll2string(buf,sizeof(buf),(long)obj->ptr);
        if (_addReplyToBuffer(c,buf,len) != C_OK)
            _addReplyProtoToList(c,buf,len);
    } else {
        serverPanic("Wrong obj->encoding in addReply()");
    }
}
```

##### prepareClientToWrite

`prepareClientToWrite`函数，在每次添加数据到output buffer之前都会调用，判断是真的需要需要发送数据给客户端 c。

```cpp
#define CLIENT_LUA 			  	 (1<<8)		/* This is a non connected client used by Lua */ //假的客户端
#define CLIENT_REPLY_OFF 		 (1<<22)	/* Don't send replies to client. */				 // 不发送
#define CLIENT_REPLY_SKIP 		 (1<<24)  	/* Don't send just this reply. */				 // 不发送 
#define CLIENT_MASTER 			 (1<<1)  	/* This client is a master */					
#define CLIENT_MASTER_FORCE_REPLY (1<<13)   /* Queue replies even if is master */				

int prepareClientToWrite(client *c) {
    /* If it's the Lua client we always return ok without installing any
     * handler since there is no socket at all. */
    if (c->flags & (CLIENT_LUA|CLIENT_MODULE)) return C_OK;		// ???

    /* CLIENT REPLY OFF / SKIP handling: don't send replies. */ 
    if (c->flags & (CLIENT_REPLY_OFF|CLIENT_REPLY_SKIP)) return C_ERR; // 不发送

    /* Masters don't receive replies, unless CLIENT_MASTER_FORCE_REPLY flag
     * is set. */
    if ((c->flags & CLIENT_MASTER) && !(c->flags & CLIENT_MASTER_FORCE_REPLY)) return C_ERR; 

    if (!c->conn) return C_ERR; /* Fake client for AOF loading. */

    /* Schedule the client to write the output buffers to the socket, unless
     * it should already be setup to do so (it has already pending data). */
    // 如果之前的数据已经空了，则重新注册可写事件
    if (!clientHasPendingReplies(c)) clientInstallWriteHandler(c);  // 此处的安装handler，
    														   // 就是设置个标志位 CLIENT_PENDING_WRITE

    /* Authorize the caller to queue in the output buffer of this client. */
    return C_OK;
}
```

##### clientInstallWriteHandler

设立可写标志 <font  bold color=yellow>`CLIENT_PENDING_WRITE` </font>，并将 客户端`c`加入到待写队列 `server.clients_pending_write` 中。<font  bold color=yellow>`CLIENT_PENDING_WRITE` </font>标志位在下面三个函数中被取消：
+ `unlinkClient`
+ `handleClientsWithPendingWrites`
+ `handleClientsWithPendingWritesUsingThreads`
```cpp
void clientInstallWriteHandler(client *c) {
    /* Schedule the client to write the output buffers to the socket only
     * if not already done and, for slaves, if the slave can actually receive
     * writes at this stage. */
    if (!(c->flags & CLIENT_PENDING_WRITE) &&
        (c->replstate == REPL_STATE_NONE || (c->replstate == SLAVE_STATE_ONLINE && !c->repl_put_online_on_ack)))
    {
        /* Here instead of installing the write handler, we just flag the
         * client and put it into a list of clients that have something
         * to write to the socket.
         * 
         * 这里没有注册可写事件，而是设置了可写标志位：CLIENT_PENDING_WRITE，
         * 并将客户端 c 加入了待写的链表 server.clients_pending_write 中
         *
         * This way before re-entering the event loop, 
         * we can try to directly write to the client sockets avoiding a system call. 
         * We'll only really install the write handler 
         * if we'll not be able to write the whole reply at once. */
        c->flags |= CLIENT_PENDING_WRITE; 
        listAddNodeHead(server.clients_pending_write, c);
    }
}
```

##### clientHasPendingReplies

```cpp
// 发送缓冲区中是否有数据
int clientHasPendingReplies(client *c) {
    return c->bufpos || listLength(c->reply);
}
```

#### addReplySds

```cpp
// 直接添加 sds 格式的字符串到输出缓冲区，其中 s 会被释放
void addReplySds(client *c, sds s) {
    if (prepareClientToWrite(c) != C_OK) {
        /* The caller expects the sds to be free'd. */
        sdsfree(s); 
        return;
    }
    if (_addReplyToBuffer(c,s,sdslen(s)) != C_OK)
        _addReplyProtoToList(c,s,sdslen(s));
    sdsfree(s);
}
```

#### addReplyProto

```cpp
// 直接添加指定长度的字符串
void addReplyProto(client *c, const char *s, size_t len) {
    if (prepareClientToWrite(c) != C_OK) return;

    if (_addReplyToBuffer(c,s,len) != C_OK)
        _addReplyProtoToList(c,s,len);
}
```

----

上面的回复，都是不带格式的，下面是带有格式的，详情参考[RESP](https://redis.io/topics/protocol)

### addReplyErrorLength

错误类型，前缀是  **`-`**，在 `-`之后，可以加错误码，如果没有则REdis设置为`ERR`，比如`"-Error message\r\n"`

```cpp
void addReplyErrorLength(client *c, const char *s, size_t len) {
    /* If the string already starts with "-..." then the error code
     * is provided by the caller. Otherwise we use "-ERR". */
    // 如果s中没有包含错误码，则使用-ERR替代，总体格式：-ERR <errorMsg> \r\n
    if (!len || s[0] != '-') addReplyProto(c,"-ERR ",5);
    addReplyProto(c,s,len);
    addReplyProto(c,"\r\n",2);
	
    // 下面暂时不管
    /* Sometimes it could be normal that a slave replies to a master with
     * an error and this function gets called. Actually the error will never
     * be sent because addReply*() against master clients has no effect...
     * A notable example is:
     *
     *    EVAL 'redis.call("incr",KEYS[1]); redis.call("nonexisting")' 1 x
     *
     * Where the master must propagate the first change even if the second
     * will produce an error. However it is useful to log such events since
     * they are rare and may hint at errors in a script or a bug in Redis. */
    int ctype = getClientType(c);
    if (ctype == CLIENT_TYPE_MASTER || ctype == CLIENT_TYPE_SLAVE || c->id == CLIENT_ID_AOF) {
        char *to, *from;

        if (c->id == CLIENT_ID_AOF) {
            to = "AOF-loading-client";
            from = "server";
        } else if (ctype == CLIENT_TYPE_MASTER) {
            to = "master";
            from = "replica";
        } else {
            to = "replica";
            from = "master";
        }

        char *cmdname = c->lastcmd ? c->lastcmd->name : "<unknown>";
        serverLog(LL_WARNING,"== CRITICAL == This %s is sending an error "
                             "to its %s: '%s' after processing the command "
                             "'%s'", from, to, s, cmdname);
        if (ctype == CLIENT_TYPE_MASTER && server.repl_backlog &&
            server.repl_backlog_histlen > 0)
        {
            showLatestBacklog();
        }
        server.stat_unexpected_error_replies++;
    }
}
```

#### addReplyError

```cpp
void addReplyError(client *c, const char *err) {
    addReplyErrorLength(c,err,strlen(err));
}
```

#### addReplyErrorFormat

格式化错误，可以传入不定参数。

```cpp
void addReplyErrorFormat(client *c, const char *fmt, ...) {
    size_t l, j;
    va_list ap;
    va_start(ap,fmt);
    sds s = sdscatvprintf(sdsempty(),fmt,ap);
    va_end(ap);
    /* Make sure there are no newlines in the string, otherwise invalid protocol
     * is emitted. */
    l = sdslen(s);
    for (j = 0; j < l; j++) {
        if (s[j] == '\r' || s[j] == '\n') s[j] = ' ';
    }

    addReplyErrorLength(c, s, sdslen(s));
    sdsfree(s);
}
```

### addReplyStatusLength

回复客户端的简单字符串都是以 **`+`** 开始，以 `\r\n` 结束，格式是： `+Message\r\n`

```cpp
// 给客户端回
void addReplyStatusLength(client *c, const char *s, size_t len) {
    addReplyProto(c,"+",1);
    addReplyProto(c,s,len);
    addReplyProto(c,"\r\n",2);
}
```

#### addReplyStatus

```cpp
void addReplyStatus(client *c, const char *status) {
    addReplyStatusLength(c,status,strlen(status));
}

```

#### addReplyStatusFormat

能格式化的回复简单字符串。

```cpp
void addReplyStatusFormat(client *c, const char *fmt, ...) {
    va_list ap;
    va_start(ap,fmt);
    sds s = sdscatvprintf(sdsempty(),fmt,ap);
    va_end(ap);

    addReplyStatusLength(c,s,sdslen(s));
    sdsfree(s);
}
```

### addReplyDeferredLen

`addReplyDeferredLen`函数，当不知道即将要添加多少数据到缓冲区中时使用。

+ 该函数先调用 **`trimReplyUnusedTailSpace`**  裁剪掉 `c->reply`尾部节点 `tail`中多余的空间
+ 在`tail`后新建一个 `dummy node`（节点值是`NULL`），然后在这个 `dummy node` 节点后面新建节点来存储数据。
+ 结束时，调用函数  **`setDeferredAggregateLen`**， 将新添数据的行数`len`（即多少条以`\r\n`结尾的数据）存储到`dummpy node`中（除非该len太多需要另作处理）。

```cpp
void* addReplyDeferredLen(client *c) {
    /* Note that we install the write event here even if the object is not
     * ready to be sent, since we are sure that before returning to the
     * event loop setDeferredAggregateLen() will be called. */
    if (prepareClientToWrite(c) != C_OK) return NULL;

    trimReplyUnusedTailSpace(c);
    listAddNodeTail(c->reply, NULL); /* NULL is our placeholder. */
    return listLast(c->reply); 
}
```

#### trimReplyUnusedTailSpace

裁剪 `c->reply` 尾部节点`tail`中多余的空间

```CPP
void trimReplyUnusedTailSpace(client *c) {
    listNode *ln = listLast(c->reply);
    clientReplyBlock *tail = ln? listNodeValue(ln): NULL;

    /* Note that 'tail' may be NULL even if we have a tail node, becuase when
     * addReplyDeferredLen() is used */
    if (!tail) return;

    /* We only try to trim the space is relatively high (more than a 1/4 of the allocation), 
     * otherwise there's a high chance realloc will NOP.
     * Also, to avoid large memmove which happens as part of realloc, we only do
     * that if the used part is small.  */

    // 为了避免产生较多内存碎片：可用内存超过1/4 (说明,大部分内存都被使用了), 
    // 并且已使用内存小于 PROTO_REPLY_CHUNK_BYTES，那么就裁剪
    if (tail->size - tail->used > tail->size / 4 &&
        tail->used < PROTO_REPLY_CHUNK_BYTES)
    {
        size_t old_size = tail->size;
        tail = zrealloc(tail, tail->used + sizeof(clientReplyBlock)); 
        /* take over the allocation's internal fragmentation (at least for
         * memory usage tracking) */
        tail->size = zmalloc_usable(tail) - sizeof(clientReplyBlock); 
        c->reply_bytes = c->reply_bytes + tail->size - old_size;
        listNodeValue(ln) = tail;
    }
}
```

#### setDeferredAggregateLen

这是在添加数据结束时使用，将添加了数据的行数`len`设置在dummy node中

```cpp
void setDeferredAggregateLen(client *c, void *node, long length, char prefix) {
    listNode *ln = (listNode*)node; // dummpy node
    clientReplyBlock *next;			// 新添加数据的第一个节点
    char lenstr[128];

    // 回复给客户端的第一行: c len \r\n
    size_t lenstr_len = sprintf(lenstr, "%c%ld\r\n", prefix, length);

    /* Abort when *node is NULL: when the client should not accept writes
     * we return NULL in addReplyDeferredLen() */
    // node 是NULL 表示不应该发数据给客户端, 比如  prepareClientToWrite 返回 C__ERR
    if (node == NULL) return;
    serverAssert(!listNodeValue(ln));
    
    // 填充 dummy node，其值是NULL，现在为其分配值
    /* Normally we fill this dummy NULL node, added by addReplyDeferredLen(),
     * with a new buffer structure containing the protocol needed to specify
     * the length of the array following. However sometimes when there is
     * little memory to move, we may instead remove this NULL node, and prefix
     * our protocol in the node immediately after to it, in order to save a
     * write(2) syscall later. Conditions needed to do it:
     *
     * - The next node is non-NULL,
     * - It has enough room already allocated
     * - And not too large (avoid large memmove) */
    if (ln->next != NULL && 
        (next = listNodeValue(ln->next)) &&
        lenstr_len <= (next->size - next->used) && // 下一个节点可用大小可容纳 lenser
        next->used < PROTO_REPLY_CHUNK_BYTES * 4)   
    {
        // 1 那么就可以将 lenstr 插入 next->buf 的头部
        memmove(next->buf + lenstr_len, next->buf, next->used);
        memcpy(next->buf, lenstr, lenstr_len);
        next->used += lenstr_len;
        listDelNode(c->reply, ln); // 将 dummy node删除，那么ln->next的上一个节点是
        						 // 被 trimReplyUnusedTailSpace 裁剪过的节点
    } else {
        /* Create a new node */
        // 2 否则就创建一个新的缓冲区来存储 len_str，
        clientReplyBlock *buf = zmalloc(lenstr_len + sizeof(clientReplyBlock));
        /* Take over the allocation's internal fragmentation */
        buf->size = zmalloc_usable(buf) - sizeof(clientReplyBlock);
        buf->used = lenstr_len;
        memcpy(buf->buf, lenstr, lenstr_len);
        listNodeValue(ln) = buf; // 让这个缓冲区成为这个dummy node的值
        c->reply_bytes += buf->size;
    }
    // 查看 c->reply 是否超过限制
    asyncCloseClientOnOutputBufferLimitReached(c);
}
```

#### setDeferredAggregateLen应用

```cpp
void setDeferredArrayLen(client *c, void *node, long length) {
    setDeferredAggregateLen(c,node,length,'*');
}

void setDeferredMapLen(client *c, void *node, long length) {
    int prefix = c->resp == 2 ? '*' : '%';
    if (c->resp == 2) length *= 2;
    setDeferredAggregateLen(c,node,length,prefix);
}

void setDeferredSetLen(client *c, void *node, long length) {
    int prefix = c->resp == 2 ? '*' : '~';
    setDeferredAggregateLen(c,node,length,prefix);
}

void setDeferredAttributeLen(client *c, void *node, long length) {
    int prefix = c->resp == 2 ? '*' : '|';
    if (c->resp == 2) length *= 2;
    setDeferredAggregateLen(c,node,length,prefix);
}

void setDeferredPushLen(client *c, void *node, long length) {
    int prefix = c->resp == 2 ? '*' : '>';
    setDeferredAggregateLen(c,node,length,prefix);
}
```

### addReplyDouble

在缓冲区中添加一个浮点数。在REdis6.0之前的格式是:

```shell
$double_num_len\r\n  # 第一行是回复的浮点数长度，以 '$'开始
double_num			# 第二行是字符串格式的浮点数
```

在REdis 6.0开始以后的格式：

````
double_num\r\n
````

```cpp
void addReplyDouble(client *c, double d) {
    if (isinf(d)) {
        /* Libc in odd systems (Hi Solaris!) will format infinite in a
         * different way, so better to handle it in an explicit way. */
        if (c->resp == 2) {
            addReplyBulkCString(c, d > 0 ? "inf" : "-inf");
        } else {
            addReplyProto(c, 
                          d > 0 ? ",inf\r\n" : ",-inf\r\n",
                          d > 0 ? 6 : 7);
        }
    } else {
        char dbuf[MAX_LONG_DOUBLE_CHARS+3], sbuf[MAX_LONG_DOUBLE_CHARS+32]; 

        int dlen, slen;
        if (c->resp == 2) {
            dlen = snprintf(dbuf,sizeof(dbuf), "%.17g", d); 
            slen = snprintf(sbuf,sizeof(sbuf), "$%d\r\n%s\r\n",dlen,dbuf); 
            addReplyProto(c,sbuf,slen);
        } else {
            dlen = snprintf(dbuf,sizeof(dbuf),",%.17g\r\n",d);
            addReplyProto(c,dbuf,dlen);
        }
    }
}
```

#### addReplyHumanLongDouble

回复客户端 `long double` 格式的字符串，也区分版本。在REdis6.0之前，是按照批量（`bulk`）回复字符串格式，即以 `$` 为前缀，格式如下：

```
/**
 *   Simple Strings prefix: "+"
     Errors         prefix: "-"
     Integers       prefix: ":"
     Bulk Strings   prefix: "$"
     Arrays         prefix: "*"
*/
```

在REdis之后，`Long double` 是以 `,` 为前缀。

```cpp
void addReplyHumanLongDouble(client *c, long double d) {
    if (c->resp == 2) {
        robj *o = createStringObjectFromLongDouble(d, 1);
        addReplyBulk(c,o);
        decrRefCount(o);
    } else {
        char buf[MAX_LONG_DOUBLE_CHARS];
        int len = ld2string(buf,sizeof(buf),d,LD_STR_HUMAN);
        addReplyProto(c,",",1);  // 前缀
        addReplyProto(c,buf,len);
        addReplyProto(c,"\r\n",2);
    }
}
```

### addReplyLongLongWithPrefix

以指定前缀回复客户端整数.

```cpp
void addReplyLongLongWithPrefix(client *c, long long ll, char prefix) {
    char buf[128];
    int len;

    /* Things like $3\r\n or *2\r\n are emitted very often by the protocol
     * so we have a few shared objects to use if the integer is small
     * like it is most of the times. */
    if (prefix == '*' && 0 <= ll && ll < OBJ_SHARED_BULKHDR_LEN) {
        // 数组格式
        addReply(c, shared.mbulkhdr[ll]); // 回复给客户端的是：*ll\r\n
        return;
    } else if (prefix == '$' && 0 <= ll && ll < OBJ_SHARED_BULKHDR_LEN) {
        // 批量字符串格式
        addReply(c, shared.bulkhdr[ll]); // 回复给客户端的是： $llr\n
        return;
    }

    // 非  * $ 类型
    buf[0] = prefix;	// 指定前缀
    len = ll2string(buf+1, sizeof(buf)-1, ll);
    buf[len+1] = '\r';
    buf[len+2] = '\n';
    addReplyProto(c,buf,len+3);
}
```

上面的 ` shared.mbulkhdr`和 `shared.bulkhdr`在 `server.c`中如下初始化，前缀分别是 `*`  和 `$`，表示数组和批量字符串回复。

```cpp
// in server.c

for (j = 0; j < OBJ_SHARED_BULKHDR_LEN; j++) {
        shared.mbulkhdr[j] = createObject(OBJ_STRING, sdscatprintf(sdsempty(), "*%d\r\n", j));  // 前缀是 *
        shared.bulkhdr[j]  = createObject(OBJ_STRING, sdscatprintf(sdsempty(), "$%d\r\n", j));  // 前缀是 $
    }
```

#### addReplyLongLong

整数前缀没有改变，一直是 `:`

```cpp
void addReplyLongLong(client *c, long long ll) {
    if (ll == 0)
        addReply(c,shared.czero);
    else if (ll == 1)
        addReply(c,shared.cone);
    else
        addReplyLongLongWithPrefix(c,ll,':');  // 整数前缀 ':'
}
```

### addReplyNull

NULL，在REdis6.0之前是回复 `$-1\r\n`，之后回复 `_\r\n`

```cpp
void addReplyNull(client *c) {
    if (c->resp == 2) {
        addReplyProto(c,"$-1\r\n",5);   // NULL，在resp2中表示为： $-1\r\n
    } else {
        addReplyProto(c,"_\r\n",3);     // resp3，_\r\n
    }
}
```

#### addReplyNullArray

NULL的数组，之前回复的是 `*-1\r\n`，现在是 `_\r\n`

```cpp
void addReplyNullArray(client *c) {
    if (c->resp == 2) {
        addReplyProto(c,"*-1\r\n",5);
    } else {
        addReplyProto(c,"_\r\n",3);
    }
}
```

### addReplyBulkLen

回复客户端的批量字符串长度，前缀是 `$`，即 `$len\r\n`

```cpp
#define OBJ_SHARED_BULKHDR_LEN 32

void addReplyBulkLen(client *c, robj *obj) {
    size_t len = stringObjectLen(obj);
    
    if (len < OBJ_SHARED_BULKHDR_LEN)
        addReply(c,shared.bulkhdr[len]);       // $
    else
        addReplyLongLongWithPrefix(c,len,'$'); // 指定 $
}
```

#### addReplyBulk

批量回复客户端，格式:

```
$len\r\n
message\r\n
```

```cpp
/* Add a Redis Object as a bulk reply */
void addReplyBulk(client *c, robj *obj) {
    addReplyBulkLen(c,obj);
    addReply(c,obj);
    addReply(c,shared.crlf);
}
```

#### addReplyBulkCBuffer

指定了长度的批量字符串，回复客户都

```cpp
void addReplyBulkCBuffer(client *c, const void *p, size_t len) {
    addReplyLongLongWithPrefix(c,len,'$');
    addReplyProto(c,p,len);
    addReply(c,shared.crlf);
}
```

#### addReplyBulkSds

sds格式的批量回复

```cpp
void addReplyBulkSds(client *c, sds s)  {
    addReplyLongLongWithPrefix(c,sdslen(s),'$');
    addReplySds(c,s);
    addReply(c,shared.crlf);
}
```

#### addReplyBulkCString

```cpp
// 将一个c风格的字符串回复给客户端
void addReplyBulkCString(client *c, const char *s) {
    if (s == NULL) {
        addReplyNull(c);
    } else {
        addReplyBulkCBuffer(c,s,strlen(s));
    }
}
```

## 发送数据

上面的 `add*()`系列函数只是将数据添加到发送缓冲区中，但是还没发送出去，真正发送数据的函数是 `writeToClient`，即可写事件的处理函数。

### writeToClient

```cpp
#define NET_MAX_WRITES_PER_EVENT (1024*64)

int writeToClient(client *c, int handler_installed) {
    ssize_t nwritten = 0, totwritten = 0;
    size_t objlen;
    clientReplyBlock *o;

    // 表示客户端缓冲区中有待发送的数据
    while(clientHasPendingReplies(c)) {
        if (c->bufpos > 0) {
            // 先发送静态缓冲区 c->buf 中的数据
            nwritten = connWrite(c->conn, c->buf+c->sentlen, c->bufpos-c->sentlen);
            if (nwritten <= 0) break;   // 发生错误则跳出循环，在循环外判断错误
            c->sentlen += nwritten;     // 已经发送的
            totwritten += nwritten;     // 待发送的

            /* If the buffer was sent, set bufpos to zero to continue with
             * the remainder of the reply. */
            if ((int)c->sentlen == c->bufpos) {
                c->bufpos = 0;
                c->sentlen = 0;
            }
        } 
        else {
            // 再发送动态缓冲区中的数据:
            //      从头部节点开始发送依次发送,每次发送完一个节点的数据,就将其从 c->reply 中删除
            //      当 c->reply 中的所有数据都发送完毕,此时 c->reply_bytes 也应该是0
            o = listNodeValue(listFirst(c->reply));
            objlen = o->used;   //当前节点的待发送数据长度

            if (objlen == 0) {
                c->reply_bytes -= o->size;
                listDelNode(c->reply, listFirst(c->reply));
                continue;
            }

            nwritten = connWrite(c->conn, o->buf + c->sentlen, objlen - c->sentlen);
            if (nwritten <= 0) break;
            c->sentlen += nwritten;
            totwritten += nwritten;

            /* If we fully sent the object on head go to the next one */
            if (c->sentlen == objlen) {
                c->reply_bytes -= o->size;
                listDelNode(c->reply,listFirst(c->reply));
                c->sentlen = 0;
                /* If there are no longer objects in the list, we expect
                 * the count of reply bytes to be exactly zero. */
                if (listLength(c->reply) == 0)
                    serverAssert(c->reply_bytes == 0);
            }
        }

        /** Note that we avoid to send more than NET_MAX_WRITES_PER_EVENT bytes, 
         * in a single threaded server it's a good idea to serve other clients as well, 
         * even if a very large request comes from super fast link that is always able to accept data 
         * (in real world scenario think about 'KEYS *' against the loopback interface).
         *
         * However if we are over the maxmemory limit we ignore that and
         * just deliver as much data as it is possible to deliver.
         *
         * Moreover, we also send as much as possible if the client is
         * a slave or a monitor (otherwise, on high-speed traffic, the
         * replication/output buffer will grow indefinitely) */
        // 避免单次发送过多的数据
        if (totwritten > NET_MAX_WRITES_PER_EVENT &&
            (server.maxmemory == 0 || zmalloc_used_memory() < server.maxmemory) && 
            !(c->flags & CLIENT_SLAVE))
        {
            break;
        } 
    } // while-end

    server.stat_net_output_bytes += totwritten; // 发送的总字节
    // 如果发生错误
    if (nwritten == -1) {
        // 这个错误是无伤大雅性质，则忽略
        if (connGetState(c->conn) == CONN_STATE_CONNECTED) {
            nwritten = 0;
        } else {
            // 否则释放客户端
            serverLog(LL_VERBOSE, "Error writing to client: %s", connGetLastError(c->conn));
            freeClientAsync(c);
            return C_ERR;
        }
    }

    if (totwritten > 0) {
        /* For clients representing masters we don't count sending data
         * as an interaction, since we always send REPLCONF ACK commands
         * that take some time to just fill the socket output buffer.
         * We just rely on data / pings received for timeout detection. */
        if (!(c->flags & CLIENT_MASTER)) c->lastinteraction = server.unixtime;
    }
    
    // 如果客户端缓冲区已经没有数据了
    if (!clientHasPendingReplies(c)) {
        c->sentlen = 0;
        /* Note that writeToClient() is called in a threaded way, but
         * adDeleteFileEvent() is not thread safe: however writeToClient()
         * is always called with handler_installed set to 0 from threads
         * so we are fine. */
        
        if (handler_installed) connSetWriteHandler(c->conn, NULL);   // handler_installed 设置为1，
        													    // 则在 output buffer 发送完时，
        													    // 取消注册可写事件，防止busy loop

        /* Close connection after entire reply has been sent. */
        // 缓冲区中的数据都被发送了，并且设置了标志位 CLIENT_CLOSE_AFTER_REPLY， 就关闭客户端
        if (c->flags & CLIENT_CLOSE_AFTER_REPLY) {
            freeClientAsync(c);
            return C_ERR;
        }
    }

    return C_OK;
}
```

上面 标志位<font color=yellow> CLIENT_CLOSE_AFTER_REPLY</font>，只是在客户端因为某些指令或者问题要关闭时才会设置。正好在设置了之后，发送缓冲区中的数据发送完毕，则直接调用 `freeClientAsync` 关闭客户端。

#### sendReplyToClient

用于当直接调用 `writeToClient` 函数，没有将发送缓冲区中数据发送完毕时，注册可写事件， `sendReplyToClient`作为此时的可写事件处理函数，因为这次会将发送缓冲区中的数据发送完，需要取消注册可写事件，因此 `writeToClient`的第二个参数设置为1。

```cpp
/* Write event handler. Just send data to the client. */
void sendReplyToClient(connection *conn) {
    client *c = connGetPrivateData(conn);
    writeToClient(c,1);
}
```

#### handleClientsWithPendingWrites

函数 `handleClientsWithPendingWrites`，每次在 `epoll_wait` 之前的 `beforeSleep()` 中执行。这个函数是主动行为，即每次在 `epoll_wait`之前都会先发送一次数据。如果没有发送完，则注册可写事件，并设置回调函数为  `sendReplyToClient`，在即将阻塞的 `epll_wait`中等待事件触发。

此外，标志位 <font color=yellow>CLIENT_PENDING_WRITE</font> 在函数 `clientInstallWriteHandler`中设置的，在发送完数据后取消标志位。

```cpp
int handleClientsWithPendingWrites(void) {
    listIter li;
    listNode *ln;
    int processed = listLength(server.clients_pending_write);
	
    // 遍历每个缓冲区数据待发送的客户端
    listRewind(server.clients_pending_write,&li);
    while((ln = listNext(&li))) {
        client *c = listNodeValue(ln);
        c->flags &= ~CLIENT_PENDING_WRITE;  // 去掉标志位 CLIENT_PENDING_WRITE，表示数据已经发送
        listDelNode(server.clients_pending_write, ln);	 // 将当前客户端从 
        											 // server.clients_pending_write 中删除
		
        /* If a client is protected, don't do anything,
         * that may trigger write error or recreate handler. */
        if (c->flags & CLIENT_PROTECTED) continue;

        /* Try to write buffers to the client socket. */
        // 将这个客户端的 output buffer 发送给客户端
        if (writeToClient(c,0) == C_ERR) continue;

        /* If after the synchronous writes above we still have data to
         * output to the client, we need to install the writable handler. */
        // 如果还有数据待发送,则需要注册可写事件,等待可写事件的到来
        if (clientHasPendingReplies(c)) {
            int ae_barrier = 0;
            /* For the fsync=always policy, we want that a given FD is never
             * served for reading and writing in the same event loop iteration,
             * so that in the middle of receiving the query, and serving it
             * to the client, we'll call beforeSleep() that will do the
             * actual fsync of AOF to disk. the write barrier ensures that. */
            if (server.aof_state == AOF_ON &&
                server.aof_fsync == AOF_FSYNC_ALWAYS)
            {
                ae_barrier = 1;
            }

            // 关注可写事件,并设置可写事件处理函数 sendReplyToClient
            // 在即将阻塞的 poll_wait 中等待可写事件触发
            if (connSetWriteHandlerWithBarrier(c->conn, sendReplyToClient, ae_barrier) == C_ERR) {
                freeClientAsync(c);
            }
        }
    }
    return processed;
}
```





