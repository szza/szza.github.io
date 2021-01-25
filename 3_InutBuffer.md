# InputBuffer

`client`中与输入缓冲区有关的字段如下：

```cpp
struct client { 
    // ...
    sds querybuf;           			// 存储客户端的请求数据
    size_t qb_pos;          			// querybuf中已经处理的字节数
    sds pending_querybuf;   
    size_t querybuf_peak;    			// 最大的请求数据
    int argc;               			// 当前指令的参数个数
    robj **argv;            			// 当前指令的参数
    struct redisCommand *cmd, *lastcmd;  // 当前待执行的指令以及上一个指令

    int reqtype;           				// 客户端发起的请求类型： Inline 和 MulitiBulk
    int multibulklen;    				// 表示请求类型是 multibulk 时，指令cmd参数个数 +1（加1，因为还有指令）
    long bulklen;           			// 每个参数的字节数
    // ...
  	list* clients_pending_read;  		// 待读取任务列表，用在多线程中
    // ...
};
```

## 单线程-读

### readQueryFromClient

这个函数是REdis处理可读事件的回调函数，负责读取客户端的请求数据

```cpp
client *createClient(connection *conn) {
     	// ...
       if (conn) {
           // ...
           connSetReadHandler(conn, readQueryFromClient); // 注册可读事件，可读事件的回调函数是 readQueryFromClient
           // ..
       }
}

```

`readQueryFromClient`函数，主要有如下几个步骤：

+ 基于 `postponeClientRead`函数，判断是否需要将这个读取操作延迟到子线程中执行

+ 判断此次数据是否是上一个指令的后续部分，详细可以参考后面的  `processMultibulkBuffer` 函数中分析。

  ```cpp
      if (c->reqtype == PROTO_REQ_MULTIBULK && 
          c->multibulklen && 
          c->bulklen != -1 && c->bulklen >= PROTO_MBULK_BIG_ARG)
      {
          ssize_t remaining = (size_t)(c->bulklen+2)-sdslen(c->querybuf);
  
          /* Note that the 'remaining' variable may be zero in some edge case,
           * for example once we resume a blocked client after CLIENT PAUSE. */
          if (0 < remaining  && remaining < readlen) readlen = remaining;
      }
  ```

+ 调用 `connRead`函数读取数据
  + 根据返回值及`conn`状态，判断`connRead`结果
  + 如果读取成功，且 `c->querybuf` 中数据超过限制，也要关闭客户端（<font color=red> 应用层的流量控制</font>）
  
+ 调用` processInputBuffer`函数，解析并执行指令

```CPP
void readQueryFromClient(connection *conn) {
    client *c = connGetPrivateData(conn);
    int nread, readlen;
    size_t qblen;

    /* Check if we want to read from the client later when exiting from
     * the event loop. This is the case if threaded I/O is enabled. */
    //  如果当前是在 EventLoop 线程中调用此函数，判断是否需要暂停读取数据，
    //  将任务延迟到子线程去执行
    if (postponeClientRead(c)) return;

    readlen = PROTO_IOBUF_LEN;   // 单次读取数据的最大值 16K 

    /* If this is a multi bulk request, and we are processing a bulk reply
     * that is large enough, try to maximize the probability that the query
     * buffer contains exactly the SDS string representing the object, even
     * at the risk of requiring more read(2) calls. 
     * 
     * This way the function processMultiBulkBuffer() can avoid copying buffers to 
     * create the Redis Object representing the argument. */
    // 如果之前的一个命令相关参数没有接受完，这次接着处理
    // 而且，这个只有一种请求类型，即 c->bulklen >= PROTO_MBULK_BIG_ARG
    // 此时的 c->querybuf 中的字符数不足 c->bulklen 个字节
    if (c->reqtype == PROTO_REQ_MULTIBULK && 
        c->multibulklen && 
        c->bulklen != -1 && c->bulklen >= PROTO_MBULK_BIG_ARG)
    {
        ssize_t remaining = (size_t)(c->bulklen+2)-sdslen(c->querybuf);

        /* Note that the 'remaining' variable may be zero in some edge case,
         * for example once we resume a blocked client after CLIENT PAUSE. */
        if (0 < remaining  && remaining < readlen) readlen = remaining;
    }

    qblen = sdslen(c->querybuf);                             
    if (c->querybuf_peak < qblen) c->querybuf_peak = qblen;  // 记录最大的请求量
    c->querybuf = sdsMakeRoomFor(c->querybuf, readlen);      // 将 c->querybuf 容量扩到 readlen 大小
    													// 为下面接受数据准备
    nread = connRead(c->conn, c->querybuf+qblen, readlen);   // 接受数据
    if (nread == -1) {
        if (connGetState(conn) == CONN_STATE_CONNECTED) { 
            return;
        } else {
            serverLog(LL_VERBOSE, "Reading from client: %s", connGetLastError(c->conn));
            freeClientAsync(c);
            return;
        }
    } 
    else if (nread == 0) {
        // 客户端关闭
        serverLog(LL_VERBOSE, "Client closed connection");
        freeClientAsync(c);
        return;
    } 
    else if (c->flags & CLIENT_MASTER) {
        // nread > 0 && (c->flags & CLIENT_MASTER)

        /* Append the query buffer to the pending (not applied) buffer
         * of the master. We'll use this buffer later in order to have a
         * copy of the string applied by the last command executed. */
        c->pending_querybuf = sdscatlen(c->pending_querybuf,
                                        c->querybuf+qblen,
                                        nread);
    }

    sdsIncrLen(c->querybuf, nread);         // 与 sdsMakeRoomFor 一起使用
    c->lastinteraction = server.unixtime;   // 记录交互时间
    if (c->flags & CLIENT_MASTER) 
        c->read_reploff += nread;
    server.stat_net_input_bytes += nread;   // for debug

    // 如果客户端请求量过大，导致服务器处理不过来
    // 也关闭客户端，防止恶意客户端
    if (sdslen(c->querybuf) > server.client_max_querybuf_len) {
        sds ci = catClientInfoString(sdsempty(),c), bytes = sdsempty();

        bytes = sdscatrepr(bytes, c->querybuf, 64);
        serverLog(LL_WARNING,
                  "Closing client that reached max query buffer length: %s (qbuf initial bytes: %s)", 
                   ci, bytes);
        sdsfree(ci);
        sdsfree(bytes);
        freeClientAsync(c);
        return;
    }

    /* There is more data in the client input buffer, continue parsing it
     * in case to check if there is a full command to execute. */
     processInputBuffer(c);
}
```

#### postponeClientRead

如果开启了读子线程，`postponeClientRead` 函数将暂停`EventLoop`中的读取操作并添加到任务队列  `server.clients_pending_read` 中，延迟线程中去执行。

```cpp
int postponeClientRead(client *c) {
    if (io_threads_active &&                    // 子线程启动
        server.io_threads_do_reads &&           // 存在读任务队列
        !ProcessingEventsWhileBlocked &&        // 没有阻塞
        !(c->flags & (CLIENT_MASTER|CLIENT_SLAVE|CLIENT_PENDING_READ)))
    {
        c->flags |= CLIENT_PENDING_READ;        		    // 则设置延迟读取客户端的请求
        listAddNodeHead(server.clients_pending_read, c);	// 加入到待读任务队列，放到子线程执行
        return 1;
    }

    return 0;
}
```

#### processInputBuffer

`processInputBuffer` 函数，根据请求格式调用`processInlineBuffer`  函数或 `processMultibulkBuffer` 函数来解析指令和指令参数，最后调用，最后调用  `processCommandAndResetClient` 函数执行指令。

1. 确定客户端的请求类型`c->reqtype`

    ```cpp
    /* Client request types */
    #define PROTO_REQ_INLINE   1	   // 第一个字符不是 *   内部协议	
    #define PROTO_REQ_MULTIBULK 2       // 第一个字符是 *	    RESP协议，即与客户端的通讯协议
    ```

2. 根据`c->reqtype`不同，调用不同的函数来解析输出。

    + `c->reqtype == PROTO_REQ_INLINE`，     用  `processInlineBuffer`  解析指令和参数，填充 `c->cmd`、`c->argv` 和 `c->argc`
    + `c->reqtype == PROTO_REQ_MULTIBULK`，用  `processMultibulkBuffer`  解析指令和参数，填充 `c->cmd`、`c->argv` 和 `c->argc`

3. 解析完成后

    + 如果这个客户端被加上标志位 `CLIENT_PENDING_READ`， 则此次不处理读取任务，再次加上标志位 `CLIENT_PENDING_COMMAND`， 跳`while`循环，等到函数 `handleClientsWithPendingReadsUsingThreads` 中去执行。

    + f如果没有加上标志位 `CLIENT_PENDING_READ`，则直接调用 `processCommandAndResetClient` 函数执行请求的命令。

```cpp
void processInputBuffer(client *c) {
    /* Keep processing while there is something in the input buffer */
    while(c->qb_pos < sdslen(c->querybuf)) {
        /* Return if clients are paused. */
        if (!(c->flags & CLIENT_SLAVE) && clientsArePaused()) break;

        /* Immediately abort if the client is in the middle of something. */
        // 如果客户端正在阻塞、等待服务器回复，则此时客户端的请求不处理
        // 比如，客户端在执行 BRPUSH 阻塞命令
        if (c->flags & CLIENT_BLOCKED) break;

        /* Don't process more buffers from clients that have already pending
         * commands to execute in c->argv. */
        // c->argv 中已经有待处理的指令，当前这个就不处理了
        if (c->flags & CLIENT_PENDING_COMMAND) break;

        /* Don't process input from the master while there is a busy script
         * condition on the slave. We want just to accumulate the replication
         * stream (instead of replying -BUSY like we do with other clients) and
         * later resume the processing. */
        if (server.lua_timedout && c->flags & CLIENT_MASTER) break;

        /* CLIENT_CLOSE_AFTER_REPLY closes the connection once the reply is
         * written to the client. Make sure to not let the reply grow after
         * this flag has been set (i.e. don't process more commands).
         * 
         * The same applies for clients we want to terminate ASAP.
         * 
         * 由于设置了标志位 CLIENT_CLOSE_AFTER_REPLY | CLIENT_CLOSE_ASAP 
         * 就不应再继续处理客户端的请求，而是尽快地关闭客户端
         */
        if (c->flags & (CLIENT_CLOSE_AFTER_REPLY|CLIENT_CLOSE_ASAP)) break;

        /* Determine request type when unknown. */
        /*************************** 确定请求类型 ************************/
        if (!c->reqtype) {
            if (c->querybuf[c->qb_pos] == '*') {
                c->reqtype = PROTO_REQ_MULTIBULK; 
            } else {
                c->reqtype = PROTO_REQ_INLINE;    
            }
        }
        
		/*********************** 不同的类型，不同的解析 ********************/
        if (c->reqtype == PROTO_REQ_INLINE) {   
            if (processInlineBuffer(c) != C_OK)     // 只有一行，处理完成 
                break;  

            /* If the Gopher mode and we got zero or one argument, process
             * the request in Gopher mode. */
            // ??? 
            if (server.gopher_enabled &&
                ((c->argc == 1 && 
                 ((char*)(c->argv[0]->ptr))[0] == '/') || c->argc == 0))
            {
                processGopherRequest(c);
                resetClient(c);
                c->flags |= CLIENT_CLOSE_AFTER_REPLY; 	// 尽快关闭
                break;
            }
        } else if (c->reqtype == PROTO_REQ_MULTIBULK) {
            if (processMultibulkBuffer(c) != C_OK) 		// 多行处理，出错忽略此次行为
                break;
        } else {
            serverPanic("Unknown request type");
        }
		
        /*********************** 处理解析结果 ********************/
        /* Multibulk processing could see a <= 0 length. */
        // 没有指令
        if (c->argc == 0) {
            resetClient(c);				// 重置客户端状态
        } else {
            /* If we are in the context of an I/O thread, 
             * we can't really execute the command here. 
             * 
             * All we can do is to flag the client 
             * as one that needs to process the command.
             *  
             * 如果开启了子线程，那么不能就地执行指令，
             * 应该将客户端标记为需要处理，让子线程去处理
             */
            if (c->flags & CLIENT_PENDING_READ) {
                c->flags |= CLIENT_PENDING_COMMAND;  // 打上标志，跳出while循环
                break; 
            }

            /* We are finally ready to execute the command. */
            // 执行指令
            if (processCommandAndResetClient(c) == C_ERR) {
                /* If the client is no longer valid, we avoid exiting this
                 * loop and trimming the client buffer later. So we return
                 * ASAP in that case. */
                return;
            }
        }
    } // while-end

    /* Trim to pos */
    // sdsrange 裁剪掉已读数据
    if (c->qb_pos) {
        sdsrange(c->querybuf, c->qb_pos, -1);
        c->qb_pos = 0;
    }
}
```

#### processInlineBuffer

 `processInlineBuffer` 函数用于内部交流，不涉及服务器和客户端的通讯。

#### processMultibulkBuffer

`processMultibulkBuffer` 函数用于处理RESP协议，接收到的数据格式基本如下：数据与数据之间都是以 `\r\n` 间隔。整个数据的前缀是 `*`， 表示下面有`multibulklen`个参数，下面每个参数前都有个以 `$` 为前缀的字符串，标志着该参数的字节数`bulklen`。

```cpp
   *总的个数				// multibulklen
   $字节个数1\r\n数据1\r\n
   $字节个数2\r\n数据2\r\n
   $字节个数3\r\n数据3\r\n
```
`processMultibulkBuffer` 函数，每次最多处理 `multibulklen`个参数，刚好是一条指令及其参数。但是由于单次最多读取`PROTO_IOBUF_LEN`字节，可能某一条指令的参数很大，无法一次处理就需要等到下次接着处理。整个过程如下：

1. 先解析出 `multibulklen`

   ```cpp
   if (c->multibulklen == 0) { 
   	// 解析multibulklen的工作
   }
   ```

   那什么时候 `c->multibulklen!=0`? ，看下面。

2. 尝试解析 `multibulklen` 个参数，第一个是指令名，第二个开始是参数名（位方便表达，都统称参数）

3. 解析参数的字节数`bulklen`。

   + 如果 `bulklen`较大， `c->querybuf`剩余字节数不足`bulklen`，则如下处理：

     ```cpp
      if (sdslen(c->querybuf)-c->qb_pos <= (size_t)ll+2) {
         sdsrange(c->querybuf, c->qb_pos, -1);
         c->qb_pos = 0;
         /* Hint the sds library about the amount of bytes this string is
          * going to contain. */
         // 使 c->querybuf 扩容到 ll+2，接受数据
         c->querybuf = sdsMakeRoomFor(c->querybuf,ll+2);
     }
     ```

     那么就等待下次可读事件触发将剩余数据发送过来。本次解析在下面的判断中直接`break`，`processMultibulkBuffer`函数返回`C_ERR`.

     ```cpp
       if (sdslen(c->querybuf)-c->qb_pos < (size_t)(c->bulklen+2)) {
             /* Not enough data (+2 == trailing \r\n) */
             break;
         } 
     ```

     当终于，一个指令的所有参数都接受完整后，使用下面的分支来创建对象

     ```cpp
     if (c->qb_pos == 0 &&
         c->bulklen >= PROTO_MBULK_BIG_ARG &&
         sdslen(c->querybuf) == (size_t)(c->bulklen+2))
     {
         c->argv[c->argc++] = createObject(OBJ_STRING, c->querybuf);
         sdsIncrLen(c->querybuf, -2);    /* remove CRLF */
         /* Assume that if we saw a fat argument we'll see another one likely... */
         c->querybuf = sdsnewlen(SDS_NOINIT, c->bulklen+2);
         sdsclear(c->querybuf);
     } 
     ```

   + 如果 `bulklen` 正常，则使用下面的函数来创建参数

     ```cpp
     c->argv[c->argc++] = createStringObject(c->querybuf+c->qb_pos, c->bulklen); 
     c->qb_pos += c->bulklen+2;  // 移动到下一行
     ```

4. 重复第3步骤，直至解析完 `multibulklen` 个参数，即完成的一个指令解析完，返回`C_OK`。

   ```cpp
    if (c->multibulklen == 0) return C_OK;
   ```

整个函数执行流程如下：

```cpp
int processMultibulkBuffer(client *c) {
    char *newline = NULL;
    int ok;
    long long ll;
	
    /*********************** 处理 multibulklen ********************/
    if (c->multibulklen == 0) {
        /* The client should have been reset */
        serverAssertWithInfo(c, NULL, c->argc == 0);
        
        /* Multi bulk length cannot be read without a \r\n */
        // 定位到 \r 位置
        newline = strchr(c->querybuf+c->qb_pos,'\r');
        if (newline == NULL) {
            // 不存在则无法处理
            if (sdslen(c->querybuf)-c->qb_pos > PROTO_INLINE_MAX_SIZE) {
                addReplyError(c,"Protocol error: too big mbulk count string");
                setProtocolError("too big mbulk count string",c);
            }
            return C_ERR;
        }
		

        /* Buffer should also contain \n */
        // querybuf------------qb_pos----------------\r\n
        //        ^               ^                   ^
        // if (newline-c->querybuf > sdslen(c->querybuf)-2))
        // 即，在在\r前面至少有两个字符, 比如 *2\r\n
        if (newline-(c->querybuf+c->qb_pos) > (ssize_t)(sdslen(c->querybuf)-c->qb_pos-2))
            return C_ERR;

        /* We know for sure there is a whole line since newline != NULL,
         * so go ahead and find out the multi bulk length. */
        // 第一个字符，即前缀应该是 *
        serverAssertWithInfo(c,NULL,c->querybuf[c->qb_pos] == '*');
        // 获取 * 之后的数字
        ok = string2ll(c->querybuf+1+c->qb_pos, newline-(c->querybuf+1+c->qb_pos), &ll);
        if (!ok || ll > 1024*1024) {
            addReplyError(c,"Protocol error: invalid multibulk length");
            setProtocolError("invalid mbulk count",c);
            return C_ERR;
        }

        c->qb_pos = (newline-c->querybuf)+2; // 将 qb_bos 移动到下一行首字节 

        if (ll <= 0) return C_OK;

        c->multibulklen = ll;               // 此次命令参数的个数

        /* Setup argv array on client structure */
        if (c->argv) zfree(c->argv);
        c->argv = zmalloc(sizeof(robj*)*c->multibulklen);
    } 

    /*********************下面具体解析 multibulklen 个行 ********************/
    
    serverAssertWithInfo(c,NULL,c->multibulklen > 0);
    while(c->multibulklen) {
        
        /******************* 解析每一个参数的字节数 bulklen ************/
        /* Read bulk length if unknown */
        if (c->bulklen == -1) {
            // 定位到 \r 位置
            newline = strchr(c->querybuf+c->qb_pos,'\r');
            if (newline == NULL) {
                // 某一行太长了，数据过多，没有\r\n
                if (sdslen(c->querybuf)-c->qb_pos > PROTO_INLINE_MAX_SIZE) {
                    addReplyError(c, "Protocol error: too big bulk count string");
                    setProtocolError("too big bulk count string",c);
                    return C_ERR;
                }
                break;
            }

            /* Buffer should also contain \n */
            // 理由同上
            if (newline-(c->querybuf+c->qb_pos) > (ssize_t)(sdslen(c->querybuf)-c->qb_pos-2))
                break;

            // 每一行的第一个字符是 $
            if (c->querybuf[c->qb_pos] != '$') {
                addReplyErrorFormat(c, 
                                    "Protocol error: expected '$', got '%c'",
                                     c->querybuf[c->qb_pos]);
                setProtocolError("expected $ but got something else",c);
                return C_ERR;
            }
			
            // 解析前缀 $ 之后的数字
            ok = string2ll(c->querybuf+c->qb_pos+1, newline-(c->querybuf+c->qb_pos+1), &ll);
            if (!ok || ll < 0 || ll > server.proto_max_bulk_len) {
                addReplyError(c,"Protocol error: invalid bulk length");
                setProtocolError("invalid bulk length",c);
                return C_ERR;
            }

            c->qb_pos = newline-c->querybuf+2;  // 移动到下一行
		    
            if (ll >= PROTO_MBULK_BIG_ARG) {	// #define PROTO_MBULK_BIG_ARG     (1024*32)
                /* If we are going to read a large object from network
                 * try to make it likely that it will start at c->querybuf
                 * boundary so that we can optimize object creation
                 * avoiding a large copy of data.
                 *
                 * But only when the data we have not parsed is less than
                 * or equal to ll+2. If the data length is greater than
                 * ll+2, trimming querybuf is just a waste of time, because
                 * at this time the querybuf contains not only our bulk. */
                
				// ll 表征参数的字节数，但是如果 sdslen(c->querybuf)-c->qb_pos <= (size_t)ll+2  
                  // 则剩余可读字节不足了 
                if (sdslen(c->querybuf)-c->qb_pos <= (size_t)ll+2) {
                    sdsrange(c->querybuf, c->qb_pos, -1);
                    c->qb_pos = 0;
                    /* Hint the sds library about the amount of bytes this string is
                     * going to contain. */
                    // 使 c->querybuf 扩容到 ll+2，接受数据
                    c->querybuf = sdsMakeRoomFor(c->querybuf,ll+2);
                }
            }

            c->bulklen = ll;
        }
			
       /******************* 解析 bulklen 表征的参数 ************/
        /* Read bulk argument */
        // 上面的 if (c->querybuf)-c->qb_pos <= (size_t)ll+2) 成立
        //  后面的剩余可解析空间不足，则break，
        // 下次可读事件到来,则 c->bulklen != -1, 直接跳过 if(bulklen ==-1)
        if (sdslen(c->querybuf)-c->qb_pos < (size_t)(c->bulklen+2)) {
            /* Not enough data (+2 == trailing \r\n) */
            break;
        } 

        /* Optimization: if the buffer contains JUST our bulk element
            * instead of creating a new object by *copying* the sds we
            * just use the current sds string. */
        // 创建对象
        if (c->qb_pos == 0 &&
            c->bulklen >= PROTO_MBULK_BIG_ARG &&
            sdslen(c->querybuf) == (size_t)(c->bulklen+2))
        {
            c->argv[c->argc++] = createObject(OBJ_STRING, c->querybuf);
            sdsIncrLen(c->querybuf, -2);    /* remove CRLF */
            /* Assume that if we saw a fat argument we'll see another one likely... */
            c->querybuf = sdsnewlen(SDS_NOINIT, c->bulklen+2);
            sdsclear(c->querybuf);
        } else {
            c->argv[c->argc++] = createStringObject(c->querybuf+c->qb_pos, c->bulklen); 
            c->qb_pos += c->bulklen+2;  // 移动到下一行
        }
        c->bulklen = -1;    // 这一个参数解析完毕

        c->multibulklen--;  // 少一个参数
    } 

    /* We're done when c->multibulk == 0 */
    if (c->multibulklen == 0) return C_OK;

    /* Still not ready to process the command */
    return C_ERR;
}
```

#### processCommandAndResetClient

+ `processCommandAndResetClient` 函数调用  `processCommand` 函数执行客户端请求的命令。
+ `commandProcessed` 一些校验和重置工作

```cpp
int processCommandAndResetClient(client *c) {
    int deadclient = 0;
    server.current_client = c;

    // processCommand 中执行指令
    if (processCommand(c) == C_OK) {
        commandProcessed(c);
    }

    if (server.current_client == NULL) deadclient = 1;
    server.current_client = NULL;
    /* freeMemoryIfNeeded may flush slave output buffers. This may
     * result into a slave, that may be the active client, to be
     * freed. */
    return deadclient ? C_ERR : C_OK;
}
```

## 多线程-读

### handleClientsWithPendingReadsUsingThreads

`handleClientsWithPendingReadsUsingThreads` 函数是在 `beforeSleep`中执行，用于将  `server.clients_pending_read` 中的读任务分到各个线程去执行，最后由主线程来执行之前加上标志位<font color=yellow> `CLIENT_PENDING_COMMAND`</font>的读任务。5个基本步骤流程如下：

1. 是否先启动读线程：第一个判断条件`server.io_threads_do_reads`是个`bool`变量，也是在`conf.c`中配置，默认是0，因此默认下，不使用多线程读取操作。
    ```cpp
      createBoolConfig("io-threads-do-reads", 
                            NULL, 
                            IMMUTABLE_CONFIG, 
                            server.io_threads_do_reads, 0, 
                            NULL, NULL), /* Read + parse from threads? */
    ```
    
2. 将任务列表 `server.clients_pending_read` 中所有的任务分发到各个线程

3. 先让子线程去执行任务

4. 主线程去执行任务，并等待子线程完成任务

5. 执行具有标志位<font color=yellow> `CLIENT_PENDING_COMMAND`</font>的任务

整个代码如下：

```cpp
int handleClientsWithPendingReadsUsingThreads(void) {
    if (!io_threads_active || !server.io_threads_do_reads) 
        return 0;

    int processed = listLength(server.clients_pending_read);   // 主线程中添加的
    if (processed == 0) return 0;

    if (tio_debug) printf("%d TOTAL READ pending clients\n", processed);

    /* Distribute the clients across N different lists. */
    // 将所有待读的任务以轮询的方式分发到  server.io_threads_num 个线程中
    listIter li;
    listNode *ln;
    listRewind(server.clients_pending_read,&li);
    int item_id = 0;
    while((ln = listNext(&li))) {
        client *c = listNodeValue(ln);
        int target_id = item_id % server.io_threads_num;
        listAddNodeTail(io_threads_list[target_id], c);
        item_id++;
    }

    /* Give the start condition to the waiting threads, by setting the
     * start condition atomic var. */
    io_threads_op = IO_THREADS_OP_READ;
    for (int j = 1; j < server.io_threads_num; j++) {
        int count = listLength(io_threads_list[j]);
        io_threads_pending[j] = count;
    }

    /*********子线程开始运行， 下面运行主线程的任务*******/
    /* Also use the main thread to process a slice of clients. */
    listRewind(io_threads_list[0],&li);
    while((ln = listNext(&li))) {
        client *c = listNodeValue(ln);
        readQueryFromClient(c->conn);
    }
    listEmpty(io_threads_list[0]);

    /***************等待所有线程执行完任务**************/
    /* Wait for all the other threads to end their work. */

    while(1) {
        unsigned long pending = 0;
        for (int j = 1; j < server.io_threads_num; j++)
            pending += io_threads_pending[j];
        if (pending == 0) break;
    }

    if (tio_debug) printf("I/O READ All threads finshed\n");

    /*************** 所有线程任务到此都执行完毕 ************/

    /**** 下面执行在EventLoop线程中设置了延迟处理的任务 ****/

    /* Run the list of clients again to process the new buffers. */
    // 那些在 eventloop 线程中设置了 CLIENT_PENDING_READ 标志位的任务，
    // 放到此处处理
    while(listLength(server.clients_pending_read)) {
        ln = listFirst(server.clients_pending_read);
        client *c = listNodeValue(ln);
        c->flags &= ~CLIENT_PENDING_READ;
        listDelNode(server.clients_pending_read, ln);

        if (c->flags & CLIENT_PENDING_COMMAND) {
            c->flags &= ~CLIENT_PENDING_COMMAND;
            if (processCommandAndResetClient(c) == C_ERR) {
                /* If the client is no longer valid, we avoid
                 * processing the client later. So we just go
                 * to the next. */
                continue;
            }
        }
        processInputBuffer(c);
    }
    return processed;
}
```





