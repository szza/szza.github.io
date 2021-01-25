# One Loop + Thread Pool

REdis6.0中加入多线程，但这不是`One Loop per Thread`模型，而是`One Loop + Thread Pool`模型，即增加了一个线程池来处理任务。 

在redisServer中，`io_threads_num`字段定义了REdis的线程数，

```cpp
struct redisServer {
    // ...
    int io_threads_num;    /* Number of IO threads to use. */
    // ...
};
```

REdis6.0默认还是单线程，可以在配置文件`config.c`中修改，REdis6.0的线程数上限是128。

```cpp
createIntConfig("io-threads", 
                NULL, 
                IMMUTABLE_CONFIG, 
                1, 128, server.io_threads_num, 1,  // 最后一个数字设置线程数，默认是单线程
                INTEGER_CONFIG, 
                NULL, NULL), 
```

### 线程变量

```cpp
// in networking.c 
int tio_debug = 1;				// only for debug

#define IO_THREADS_MAX_NUM 128   // IO_threads_max_num
#define IO_THREADS_OP_READ  0    // io_threads_op_read
#define IO_THREADS_OP_WRITE 1    // io_threads_op_write

pthread_t               io_threads[IO_THREADS_MAX_NUM];			 // 存储线程tid
pthread_mutex_t         io_threads_mutex[IO_THREADS_MAX_NUM];	  
_Atomic unsigned long   io_threads_pending[IO_THREADS_MAX_NUM];	  // 每个线程待处理的任务数，实现同步关系
int                     io_threads_active;  	 				// 线程是否启动了
int                     io_threads_op;     		 				// 主线程写，子线程读io_threads_op

list* io_threads_list[IO_THREADS_MAX_NUM];  					// 待处理的客户端任务列表
```

### beforeSleep

`beforeSleep`函数，是在EventLoop中进入阻塞之前调用（`EventLoop`怎么运转可参考[EventLoop](./1_EventLoop.md)小节描述）。每次在进入阻塞之前，都会先执行 `handleClientsWithPendingReadsUsingThreads` 和 `handleClientsWithPendingWritesUsingThreads`，使得子线程在阻塞期间也能正常运行。

```cpp
void beforeSleep(struct aeEventLoop *eventLoop)  { 
	// ...
	/* We should handle pending reads clients ASAP after event loop. */
    handleClientsWithPendingReadsUsingThreads();
    // ...
    /* Handle writes with pending output buffers. */
    handleClientsWithPendingWritesUsingThreads();
    // ...
}
```

#### handleClientsWithPendingReadsUsingThreads

详细分析见[InputBuffer](./3_InputBuffer.md)

#### handleClientsWithPendingWritesUsingThreads

在正式进入线程部分之前，先介绍下 `handleClientsWithPendingWritesUsingThreads`函数，因为它作为主线程中的生产者，将任务分发到子线程执行。

+ 在单线程模式下，`handleClientsWithPendingWritesUsingThreads`函数就是个 `handleClientsWithPendingWrites`的wrapper。
+ 在多线程模式下，有如下6个基本步骤：
  1. 将`server.clients_pending_write`中待处理的客户端，按照**轮询**的方式分发到 `server.io_threads_num`个线程的任务列表  `io_threads_list[id]` 中
  2. 设置原子变量  **`io_threads_op`** 为写操作，并将每个子线程的任务数记录到 `io_threads_pending[id]`中
  3. 第2步骤设置完，子线程可以去执行了（如何实现线程间的同步，见后文分析）
  4. 主线程去执行自己任务列表 `io_threads_list[0]` 中的任务
  5. 等待所有的子线程完成写任务
  6. 如果，还要某个客户端的output buffer中还有数据，则再注册可写事件，并设置写回调函数为 `sendReplyToClient`

整个函数流程如下：

```cpp
// 客户端的可写事件, 在 beforeSleep 中处理
int handleClientsWithPendingWritesUsingThreads(void) {
    int processed = listLength(server.clients_pending_write); // 待处理的客户端数量
    if (processed == 0) return 0; /* Return ASAP if there are no clients. */

    /* If I/O threads are disabled or we have few clients to serve, don't
     * use I/O threads, but thejboring synchronous code. */
    // 单线程模式，直接调用 handleClientsWithPendingWrites()
    if (server.io_threads_num == 1 || stopThreadedIOIfNeeded()) {
        return handleClientsWithPendingWrites();
    }

    /* Start threads if needed. */
    // 没有启动线程，启动线程
    // 能启动线程的原因见下文分析
    if (!io_threads_active) startThreadedIO(); 

    if (tio_debug) printf("%d TOTAL WRITE pending clients\n", processed);

    /* Distribute the clients across N different lists. */
    // 在主线程中，按照轮询的方式将任务分发到主线程和子线程的任务列表 io_threads_list[id]
    listIter li;
    listNode *ln;
    listRewind(server.clients_pending_write, &li);
    int item_id = 0;
    while((ln = listNext(&li))) {
        client *c = listNodeValue(ln);
        c->flags &= ~CLIENT_PENDING_WRITE;
        int target_id = item_id % server.io_threads_num;  // 采用轮询的方式选择子线程来服务这个客户端
        listAddNodeTail(io_threads_list[target_id], c);
        item_id++;
    }
	
    // 分发完毕
  
    /****************************子线程处理**************************/

    /* Give the start condition to the waiting threads, by setting the start condition atomic var. */
    // 设置原子变量 io_threads_op 为写模式，使写线程工作
    io_threads_op = IO_THREADS_OP_WRITE;
    
    // 计算每个子线程需要处理的客户端数量，存放在 io_threads_pending[j]
    // 这个数量，即上面轮询分发的
    for (int j = 1; j < server.io_threads_num; j++) {
        int count = listLength(io_threads_list[j]); // 子线程的客户端数
        io_threads_pending[j] = count;
    }

    /******上面两步设置完, 子线程才会去执行（原因见下面线程函数分析）****/

    /* Also use the main thread to process a slice of clients. */
    // 主线程也执行一部分任务，这部分任务是也是上面轮询方式分发的
    listRewind(io_threads_list[0], &li);
    while((ln = listNext(&li))) {
        client *c = listNodeValue(ln);
        writeToClient(c,0);			
    }
    listEmpty(io_threads_list[0]);	// 执行完

    /***********下面while(1)中，等待所有的子线程都处理完任务***********/
    /* Wait for all the other threads to end their work. */
    while(1) {
        unsigned long pending = 0;
        for (int j = 1; j < server.io_threads_num; j++)
            pending += io_threads_pending[j];
        if (pending == 0) break; 	// 当pending ==0时，即子线程任务都执行完毕
    }
    if (tio_debug) printf("I/O WRITE All threads finshed\n");
	
    /** 如果还有数据没有发送完毕，就再次注册可写事件（原理和单线程一致）***/
    /* Run the list of clients again to install the write handler where needed. */
    listRewind(server.clients_pending_write,&li);
    while((ln = listNext(&li))) {
        client *c = listNodeValue(ln);

        /* Install the write handler if there are pending writes in some of the clients. */
        // 为还没有发送完数据的客户端，注册可写事件
        if (clientHasPendingReplies(c) && connSetWriteHandler(c->conn, sendReplyToClient) == AE_ERR)
        {
            freeClientAsync(c);
        }
    }
    
    listEmpty(server.clients_pending_write);
    return processed;
}
```

在多线程模式下，`handleClientsWithPendingWrites/ReadUsingThreads`函数运行和单线程模式下还是有区别：

+ 在单线程中，对于客户端的请求，在`beforeSleep`函数中是先运行`handleClientsWithPendingReads`，再处理 `handleClientsWithPendingWrites`，这对于客户端的简单请求可以直接回复

+ 在多线程下，第一次必须先运行  `xxx_Write_xxx`，因为先运行 `xxx_Reads_xxx`会因为第一个if判断条件不满足而直接退出

  ```cpp
  if (!io_threads_active || !server.io_threads_do_reads) return 0;
  ```

  当处理任务较少时，有可能还是使用使用单线程来处理。

在 `handleClientsWithPendingWritesUsingThreads` 函数中的第二个`if`判断分支中，`stopThreadedIOIfNeeded`函数判断当前待执行任务`server.clients_pending_write`的数量`pengding`和线程数`server.io_threads_num`之间的关系，若`stopThreadedIOIfNeeded`返回1，则继续使用单单线程 `handleClientsWithPendingWrites`

  ```cpp
      if (server.io_threads_num == 1 || stopThreadedIOIfNeeded()) {
          return handleClientsWithPendingWrites();
      }
  ```

+ 如果第一次执行`handleClientsWithPendingWritesUsingThreads`，`stopThreadedIOIfNeeded`就返回1，子线程不会启动
+ 如果非首次执行，`stopThreadedIOIfNeeded`返回1，则会停止所有的子线程，变成单线程工作

### initThreadedIO

创建并初始化  `server.io_threads_num-1`个子线程

```cpp
void initThreadedIO(void) {
    io_threads_active = 0; /* We start with threads not active. */

    /* Don't spawn any thread if the user selected a single thread:
     * we'll handle I/O directly from the main thread. */
    //  在配置文件 config.c 中修改，最大支持128个线程
    if (server.io_threads_num == 1) return;
	// 超过限制， REdis服务器无法启动
    if (server.io_threads_num > IO_THREADS_MAX_NUM) {
        serverLog(LL_WARNING,
                  "Fatal: too many I/O threads configured. The maximum number is %d.",
                   IO_THREADS_MAX_NUM);
        exit(1);
    }

    /* Spawn and initialize the I/O threads. */
    // 下面初始化主线程和 server.io_threads_num-1 个子线程
    for (int i = 0; i < server.io_threads_num; i++) {
        /* Things we do for all the threads including the main thread. */
        // 为主线程和所有的子线程都创建一个客户端链表，即任务列表
        io_threads_list[i] = listCreate();
        if (i == 0) continue; /* Thread 0 is the main thread. */ // 下面的初始化仅针对子线程
		
        /* Things we do only for the additional threads. */
        // 为所有的子线程：创建执行线程
        pthread_t tid;
        pthread_mutex_init(&io_threads_mutex[i],NULL);
        io_threads_pending[i] = 0;				 // 初始化时，每个线程的待处理任务数为0
        
        pthread_mutex_lock(&io_threads_mutex[i]); /* Thread will be stopped. */
        
        // 启动线程， 并执行在线程中执行函数IOThreadMain
        if (pthread_create(&tid, NULL, IOThreadMain, (void*)(long)i) != 0) {
            serverLog(LL_WARNING,"Fatal: Can't initialize IO thread.");
            exit(1);
        }
        io_threads[i] = tid;	// 记录tid
    }
}
```

### IOThreadMain

子线程函数入口，在 `IOThreadMain` 里与客户端进行数据发送与接受。执行流程大致如下：

1. 至多循环100w次，等待主线程将任务分配到子线程的任务列表`io_threads_list[id]`中

2. 若待处理的任务数`pending`和线程数 `server.io_threads_num`之间满足 `pending < (server.io_threads_num*2)`，则停止多线程。检测任务数和多线程之间的关系，是在定时器事件中检测的，

   ```cpp
   int serverCron(struct aeEventLoop *eventLoop, long long id, void *clientData) { 
       //....
       /* Stop the I/O threads if we don't have enough pending work. */
       stopThreadedIOIfNeeded();
       // ...
   } 
   ```

3. 若有任务可处理，则通过判断原子变量  `io_threads_op`状态，来进行相应的读写操作

   + `IO_THREADS_OP_WRITE`：发送 OutBuffer 中的数据
   + `IO_THREADS_OP_READ`：读取并处理 InputBuffer 数据

4. `io_threads_list[id]` 中的任务执行完，主线程中`while(1)` 才能跳出。

在主线程和子线程之间，是通过原子变量 `io_threads_pending[id]`实现**同步**关系：

 	1. 在主线程中，计算了每个子线程的任务数`io_threads_pending[id]`后，子线程才去执行，然后就会阻塞在whiile(1)中，等待 `io_threads_pending[id]` 都变为0。
	2. 在子线程中，while(1)循环体需要等待`io_threads_pending[id] !=0`才能向下执行。执行完任务后，清空`io_threads_pending[id]`，主线程中while(1)才会跳出。

<font color=red>关键！！！</font> 变量  <font color=yellow>`io_threads_pending` </font>是个原子变量。因此不用`mutex`即可实现同步关系，即这个是基于`lock-free`的生产-消费模式的多线程。

```cpp
void* IOThreadMain(void *myid) {
    /* The ID is the thread number (from 0 to server.iothreads_num-1), and is
     * used by the thread to just manipulate a single sub-array of clients. */
    long id = (unsigned long)myid;
    char thdname[16];				// 线程名

    snprintf(thdname, sizeof(thdname), "io_thd_%ld", id); // 为每个线程的名字
    redis_set_thread_title(thdname);
    redisSetCpuAffinity(server.server_cpulist);
	
    // 线程循环体
    while(1) {
        /* Wait for start */
        // 循环 100w次，等待当前线程有任务可处理，
        // 即 io_threads_pending[id] 不是0
        for (int j = 0; j < 1000000; j++) {
            if (io_threads_pending[id] != 0) break;
        }

        /* Give the main thread a chance to stop this thread. */
        // 循环了 100w 次仍然没有任务可以处理，
        // 可能待处理的任务较少，有可能停止本线程
        if (io_threads_pending[id] == 0) {
            pthread_mutex_lock(&io_threads_mutex[id]); 
            pthread_mutex_unlock(&io_threads_mutex[id]);
            continue;
        }

        // 有任务可处理
        serverAssert(io_threads_pending[id] != 0);

        if (tio_debug) printf("[%ld] %d to handle\n", id, (int)listLength(io_threads_list[id]));

        /* Process: note that the main thread will never touch our list
         * before we drop the pending count to 0. */
        // 开始遍历每个线程的任务列表
        listIter  li;
        listNode* ln;
        listRewind(io_threads_list[id], &li);
        while((ln = listNext(&li))) {
            client *c = listNodeValue(ln);
            // io_threads_op 是个线程间变量，主线程设置，子线程读
            if (io_threads_op == IO_THREADS_OP_WRITE) {
                writeToClient(c,0); 		 // 发送缓冲区
            } else if (io_threads_op == IO_THREADS_OP_READ) {
                readQueryFromClient(c->conn); // 从InputBuffer中处理数据
            } else {
                serverPanic("io_threads_op value is unknown");
            }
        }
        
        listEmpty(io_threads_list[id]); 	// 将任务列表清空
        io_threads_pending[id] = 0;		 	// 待处理任务数清空，让主线程能跳出 while(1) 循环

        if (tio_debug) printf("[%ld] Done\n", id);
    }
}
```

## 线程的生命周期

### 启动顺序

下面介绍三个函数，关于子线程的启动与停止。问题：主线程和子线程谁先启动？

1. 虽然主线程是在 `initServer`函数中完成初始化，但是启动还是需要等待` aeMain`函数被调用
2. 子线程是通过 `initThreadedIO()`完成创建，并开始执行子线程入口函数 `IOThreadMain`，这一切都是在`server.c`中的 `InitServerLast()`中完成

      ```cpp
      void InitServerLast() {
          bioInit();
          initThreadedIO();
          set_jemalloc_bg_thread(server.jemalloc_bg_thread);
          server.initial_memory_usage = zmalloc_used_memory();
      }
      ```

REdis服务器的主程序中

```cpp
int main(int argc, char **argv) { 
    // ... 
	initServer();		// 主线程完成初始化
    // ...	
    InitServerLast();	// 子线程完成初始化并开始运行
    // ...
    aeMain(server.el);	// 主程序启动
    aeDeleteEventLoop(server.el);
    return 0;
} 
```

因此，可以得出的结论是先运行子线程，再运行主线程。那么下面开始分析线程的生命周期。

### 生命周期

在函数 `handleClientsWithPendingWritesUsingThreads` 中使用函数`startThreadedIO`来启动线程，其关键在于 `startThreadedIO`函数中用`for`循环来逐个解锁。整个流程如下：

1.  子线程先启动，因此`initThreadedIO`函数中对每个子线程先加上锁

    ```cpp
    pthread_mutex_lock(&io_threads_mutex[i]); /* Thread will be stopped. */
    ```
    
    由于此时主线程还没启动，没有任务分发给子线程。这会导致在子线程执行函数 `IOThreadMain` 会进入下whiie(1)循环中的条件分支，并阻塞在再次加锁位置：
    
    ```cpp
    if (io_threads_pending[id] == 0) {
        pthread_mutex_lock(&io_threads_mutex[id]); 	// 阻塞于此
        pthread_mutex_unlock(&io_threads_mutex[id]);
        continue;
    }
    ```
    
2. 当主线程启动时，`handleClientsWithPendingWritesUsingThreads`函数第一次会调用 `startThreadedIO`函数

   ```cpp
   if (server.io_threads_num == 1 || stopThreadedIOIfNeeded()) {
       return handleClientsWithPendingWrites();
   }
   
   // 不满足上面的if分支，才会启动子线程
   if (!io_threads_active) startThreadedIO(); 
   ```

   在 `startThreadedIO` 函数的`for`循环会对每个子线程依次解锁 

   ```cpp
   for (int j = 1; j < server.io_threads_num; j++)
       pthread_mutex_unlock(&io_threads_mutex[j]); 
   ```

   此时，使得子线程执行函数 `IOThreadMain`解除阻塞状态 ，能够继续运行 下去，并且往后都不会再进入 `if (io_threads_pending[id] == 0) `分支。

3.  当需要停止子线程时，在子线程停止函数`stopThreadedIO`中又对每个子线程进行了一次加锁操作，结束整个过程。

    ```cpp
    for (int j = 1; j < server.io_threads_num; j++)
        pthread_mutex_lock(&io_threads_mutex[j]);		
    ```

#### startThreadedIO

```cpp
void startThreadedIO(void) {
    if (tio_debug) {printf("S"); fflush(stdout); }
    if (tio_debug) {printf("--- STARTING THREADED IO ---\n");}
    serverAssert(io_threads_active == 0);

    for (int j = 1; j < server.io_threads_num; j++)
        pthread_mutex_unlock(&io_threads_mutex[j]);  
    io_threads_active = 1;
}
```

#### stopThreadedIOIfNeeded

判断是否要停止多线程、恢复单线程。条件即： `pending < (server.io_threads_num*2 && io_threads_active ==1 ` 。

```cpp
int stopThreadedIOIfNeeded(void) {
    int pending = listLength(server.clients_pending_write);

    /* Return ASAP if IO threads are disabled (single threaded mode). */
    if (server.io_threads_num == 1) return 1;

    if (pending < (server.io_threads_num*2)) {
        if (io_threads_active) stopThreadedIO();
        return 1;
    } else {
        return 0;
    }
}
```

#### stopThreadedIO

停止所有的子线程

```cpp
void stopThreadedIO(void) {
    /* We may have still clients with pending reads when this function
     * is called: handle them before stopping the threads. */
    // 当关闭线程IO时, 可能还有待处理的读操作
    handleClientsWithPendingReadsUsingThreads();
    if (tio_debug) { printf("E"); fflush(stdout); }
    if (tio_debug) { printf("--- STOPPING THREADED IO [R%d] [W%d] ---\n",
                          (int) listLength(server.clients_pending_read),
                          (int) listLength(server.clients_pending_write));}
    serverAssert(io_threads_active == 1);
    for (int j = 1; j < server.io_threads_num; j++)
        pthread_mutex_lock(&io_threads_mutex[j]);		
    io_threads_active = 0;
}
```

