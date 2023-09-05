---
title: Redis原理篇
date: 2022-06-28 21:25:08
categories: Redis
---

# 用户空间和内核空间

为了避免用户应用导致冲突甚至内核崩溃，用户应用与内核是分离的：
进程的寻址空间会划分为两部分：**内核空间、用户空间**
用户空间只能执行受限的命令（Ring3），而且不能直接调用系统资源，必须通过内核提供的接口来访问
内核空间可以执行特权命令（Ring0），调用一切系统资源

![image-20220705154421256](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220705154421256.png)

# IO多路复用

文件描述符（File Descriptor）：简称FD，是一个从0 开始的无符号整数，用来关联Linux中的一个文件。在Linux中，一切皆文件，例如常规文件、视频、硬件设备等，当然也包括网络套接字（Socket）。
IO多路复用：是利用单个线程来同时监听多个FD，并在某个FD可读、可写时得到通知，从而避免无效的等待，充分利用CPU资源。

![image-20220705170314983](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220705170314983.png)

阶段一：

1. 用户进程调用select，指定要监听的FD集合
2. 内核监听FD对应的多个socket
3. 任意一个或多个socket数据就绪则返回readable
4. 此过程中用户进程阻塞

阶段二：

1. 用户进程找到就绪的socket
2. 依次调用recvfrom读取数据
3. 内核将数据拷贝到用户空间
4. 用户进程处理数据

IO多路复用是利用单个线程来同时监听多个FD，并在某个FD可读、可写时得到通知，从而避免无效的等待，充分利用CPU资源。不过监听FD的方式、通知的方式又有多种实现，常见的有：

* select
* poll
* epoll

差异：
**select和poll只会通知用户进程有FD就绪，但不确定具体是哪个FD，需要用户进程逐个遍历FD来确认
epoll则会在通知用户进程FD就绪的同时，把已就绪的FD写入用户空间**

## select

select是Linux最早是由的I/O多路复用技术：

```c
// 定义类型别名 __fd_mask，本质是 long int
typedef long int __fd_mask;
/* fd_set 记录要监听的fd集合，及其对应状态 */
typedef struct {
    // fds_bits是long类型数组，长度为 1024/32 = 32
    // 共1024个bit位，每个bit位代表一个fd，0代表未就绪，1代表就绪
    __fd_mask fds_bits[__FD_SETSIZE / __NFDBITS];
    // ...
} fd_set;
// select函数，用于监听fd_set，也就是多个fd的集合
int select(
    int nfds, // 要监视的fd_set的最大fd + 1
    fd_set *readfds, // 要监听读事件的fd集合
    fd_set *writefds,// 要监听写事件的fd集合
    fd_set *exceptfds, // // 要监听异常事件的fd集合
    // 超时时间，null-用不超时；0-不阻塞等待；大于0-固定等待时间
    struct timeval *timeout
);

```

![image-20220705163424810](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220705163424810.png)

> select模式存在的问题：
> 需要将整个fd_set从用户空间拷贝到内核空间，select结束还要再次拷贝回用户空间
> select无法得知具体是哪个fd就绪，需要遍历整个fd_set
> fd_set监听的fd数量不能超过1024

## poll

poll模式对select模式做了简单改进，但性能提升不明显，部分关键代码如下：
IO流程：

1. 创建pollfd数组，向其中添加关注的fd信息，数组大小自定义
2. 调用poll函数，将pollfd数组拷贝到内核空间，转链表存储，无上限
3. 内核遍历fd，判断是否就绪
4. 数据就绪或超时后，拷贝pollfd数组到用户空间，返回就绪fd数量n
5. 用户进程判断n是否大于0
6. 大于0则遍历pollfd数组，找到就绪的fd

与select对比：

* select模式中的fd_set大小固定为1024，而pollfd在内核中采用链表，理论上无上限
* 监听FD越多，每次遍历消耗时间也越久，性能反而会下降

```c
// pollfd 中的事件类型
#define POLLIN     //可读事件
#define POLLOUT    //可写事件
#define POLLERR    //错误事件
#define POLLNVAL   //fd未打开

// pollfd结构
struct pollfd {
    int fd;     	  /* 要监听的fd  */
    short int events; /* 要监听的事件类型：读、写、异常 */
    short int revents;/* 实际发生的事件类型 */
};
// poll函数
int poll(
    struct pollfd *fds, // pollfd数组，可以自定义大小
    nfds_t nfds, // 数组元素个数
    int timeout // 超时时间
);
```

## epoll

epoll模式是对select和poll的改进，它提供了三个函数：

```c
struct eventpoll {
    //...
    struct rb_root  rbr; // 一颗红黑树，记录要监听的FD
    struct list_head rdlist;// 一个链表，记录就绪的FD
    //...
};
// 1.创建一个epoll实例,内部是event poll，返回对应的句柄epfd
int epoll_create(int size);

// 2.将一个FD添加到epoll的红黑树中，并设置ep_poll_callback
// callback触发时，就把对应的FD加入到rdlist这个就绪列表中
int epoll_ctl(
    int epfd,  // epoll实例的句柄
    int op,    // 要执行的操作，包括：ADD、MOD、DEL
    int fd,    // 要监听的FD
    struct epoll_event *event // 要监听的事件类型：读、写、异常等
);

// 3.检查rdlist列表是否为空，不为空则返回就绪的FD的数量
int epoll_wait(
    int epfd,                   // epoll实例的句柄
    struct epoll_event *events, // 空event数组，用于接收就绪的FD
    int maxevents,              // events数组的最大长度
    int timeout   // 超时时间，-1用不超时；0不阻塞；大于0为阻塞时间
);
```

![image-20220705163619695](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220705163619695.png)

select模式存在的三个问题：

* 能监听的FD最大不超过1024
* 每次select都需要把所有要监听的FD都拷贝到内核空间
* 每次都要遍历所有FD来判断就绪状态

poll模式的问题：

* poll利用链表解决了select中监听FD上限的问题，但依然要遍历所有FD，如果监听较多，性能会下降

epoll模式中如何解决这些问题的？

1. 基于epoll实例中的红黑树保存要监听的FD，理论上无上限，而且增删改查效率都非常高
2. 每个FD只需要执行一次epoll_ctl添加到红黑树，以后每次epol_wait无需传递任何参数，无需重复拷贝FD到内核空间
3. 利用ep_poll_callback机制来监听FD状态，无需遍历所有FD，因此性能不会随监听的FD数量增多而下降

## IO多路复用-事件通知机制

当FD有数据可读时，我们调用epoll_wait（或者select、poll）可以得到通知。但是事件通知的模式有两种：

* LevelTriggered：简称LT，也叫做**水平触发**。只要某个FD中有数据可读，每次调用epoll_wait都会得到通知。

* EdgeTriggered：简称ET，也叫做**边沿触发**。只有在某个FD有状态变化时，调用epoll_wait才会被通知。

  ET模式也可以重复通知：

  1. 当读取到数据后调用epoll_ctl修改内核空间监听的FD状态再去检查是否还有消息再把它添加到就绪队列中。
  2. 使用while循环不断的去读取数据，千万不能使用阻塞io，当有数据返回就处理，没有了就结束循环

举个栗子：

1. 假设一个客户端socket对应的FD已经注册到了epoll实例中

2. 客户端socket发送了2kb的数据

3. 服务端调用epoll_wait，得到通知说FD就绪

4. 服务端从FD读取了1kb数据

5. 回到步骤3（再次调用epoll_wait，形成循环）
   结果：

   √ 如果我们采用LT模式，因为FD中仍有1kb数据，则第⑤步依然会返回结果，并且得到通知
   √ 如果我们采用ET模式，因为第③步已经消费了FD可读事件，第⑤步FD状态没有变化，因此epoll_wait不会返回，数据无法读取，客户端响应超时。

LT：事件通知频率较高，会有重复通知，影响性能
ET：仅通知一次，效率高。可以基于非阻塞IO循环读取解决数据读取不完整问题

select和poll仅支持LT模式，epoll可以自由选择LT和ET两种模式

## IO多路复用-web服务流程

基于epoll模式的web服务的基本流程如图：

![image-20220705165556539](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220705165556539.png)

# 信号驱动



信号驱动IO是与内核建立SIGIO的信号关联并设置回调，当内核有FD就绪时，会发出SIGIO信号通知用户，期间用户应用可以执行其它业务，无需阻塞等待。![image-20220705170135567](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220705170135567.png)

阶段一：

1. 用户进程调用sigaction，注册信号处理函数
2. 内核返回成功，开始监听FD
3. 用户进程不阻塞等待，可以执行其它业务
4. 当内核数据就绪后，回调用户进程的SIGIO处理函数

阶段二：

1. 收到SIGIO回调信号
2. 调用recvfrom，读取
3. 内核将数据拷贝到用户空间
4. 用户进程处理数据

当有大量IO操作时，信号较多，SIGIO处理函数不能及时处理可能导致信号队列溢出，而且内核空间与用户空间的频繁信号交互性能也较低。

# 异步IO

异步IO的整个过程都是非阻塞的，用户进程调用完异步API后就可以去做其它事情，内核等待数据就绪并拷贝到用户空间后才会递交信号，通知用户进程。

![image-20220705170427999](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220705170427999.png)

阶段一：

1. 用户进程调用aio_read，创建信号回调函数
2. 内核等待数据就绪
3. 用户进程无需阻塞，可以做任何事情

阶段二：

1. 内核数据就绪

2. 内核数据拷贝到用户缓冲区

3. 拷贝完成，内核递交信号触发aio_read中的回调函数

4. 用户进程处理数据

   可以看到，异步IO模型中，用户进程在两个阶段都是非阻塞状态。

# Redis网络模型

Redis到底是单线程还是多线程？

如果仅仅聊Redis的核心业务部分（命令处理），答案是单线程
如果是聊整个Redis，那么答案就是多线程
在Redis版本迭代过程中，在两个重要的时间节点上引入了多线程的支持：
Redis v4.0：引入多线程异步处理一些耗时较旧的任务，例如异步删除命令unlink
Redis v6.0：在核心网络模型中引入 多线程，进一步提高对于多核CPU的利用率

因此，对于Redis的核心网络模型，在Redis 6.0之前确实都是单线程。是利用epoll（Linux系统）这样的IO多路复用技术在事件循环中不断处理客户端情况。

**为什么Redis要选择单线程？**
抛开持久化不谈，Redis是纯内存操作，执行速度非常快，它的性能瓶颈是网络延迟而不是执行速度，因此多线程并不会带来巨大的性能提升。
多线程会导致过多的上下文切换，带来不必要的开销
引入多线程会面临线程安全问题，必然要引入线程锁这样的安全手段，实现复杂度增高，而且性能也会大打折扣

```c
int main(
    int argc,
    char **argv
) {
    // ...
    // 初始化服务
    initServer();
    // ...
    // 开始监听事件循环
    aeMain(server.el);
    // ...
}


```

**initServer() :**

```c
void initServer(void) {
    // ...
    // 内部会调用 aeApiCreate(eventLoop)，类似epoll_create
    server.el = aeCreateEventLoop(
                    server.maxclients+CONFIG_FDSET_INCR);
    // ...
    // 监听TCP端口，创建ServerSocket，并得到FD
    listenToPort(server.port,&server.ipfd)
    // ...
    // 注册 连接处理器，内部会调用 aeApiCreate(&server.ipfd)监听FD
    createSocketAcceptHandler(&server.ipfd, acceptTcpHandler)
    // 注册 ae_api_poll 前的处理器
    aeSetBeforeSleepProc(server.el,beforeSleep);
}
    // 数据读处理器
void acceptTcpHandler(...) {
    // ...
    // 接收socket连接，获取FD
    fd = accept(s,sa,len);
    // ...
    // 创建connection，关联fd
    connection *conn = connCreateSocket();
    conn.fd = fd;
    // ... 
    // 内部调用aeApiAddEvent(fd,READABLE)，
    // 监听socket的FD读事件，并绑定读处理器readQueryFromClient
    connSetReadHandler(conn, readQueryFromClient);
}

```

aeMain:

```c
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    // 循环监听事件
    while (!eventLoop->stop) {
        aeProcessEvents(
            eventLoop, 
            AE_ALL_EVENTS|
                AE_CALL_BEFORE_SLEEP|
                AE_CALL_AFTER_SLEEP);
    }
}
int aeProcessEvents(
    aeEventLoop *eventLoop,
    int flags ){
    // ...  调用前置处理器 beforeSleep
    eventLoop->beforesleep(eventLoop);
    // 等待FD就绪，类似epoll_wait
    numevents = aeApiPoll(eventLoop, tvp);
    for (j = 0; j < numevents; j++) {
        // 遍历处理就绪的FD，调用对应的处理器
    }
}


```

![image-20220706130657802](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220706130657802.png)

```c
void readQueryFromClient(connection *conn) {
    // 获取当前客户端，客户端中有缓冲区用来读和写
    client *c = connGetPrivateData(conn);
    // 获取c->querybuf缓冲区大小
    long int qblen = sdslen(c->querybuf);
    // 读取请求数据到 c->querybuf 缓冲区
    connRead(c->conn, c->querybuf+qblen, readlen);
    // ... 
    // 解析缓冲区字符串，转为Redis命令参数存入 c->argv 数组
    processInputBuffer(c);
    // ...
    // 处理 c->argv 中的命令
    processCommand(c);
}
int processCommand(client *c) {
    // ...
    // 根据命令名称，寻找命令对应的command，例如 setCommand
    c->cmd = c->lastcmd = lookupCommand(c->argv[0]->ptr);
    // ...
    // 执行command，得到响应结果，例如ping命令，对应pingCommand
    c->cmd->proc(c);
    // 把执行结果写出，例如ping命令，就返回"pong"给client，
    // shared.pong是 字符串"pong"的SDS对象
    addReply(c,shared.pong); 
}
void addReply(client *c, robj *obj) {
    // 尝试把结果写到 c-buf 客户端写缓存区
    if (_addReplyToBuffer(c,obj->ptr,sdslen(obj->ptr)) != C_OK)
            // 如果c->buf写不下，则写到 c->reply，这是一个链表，容量无上限
            _addReplyProtoToList(c,obj->ptr,sdslen(obj->ptr));
    // 将客户端添加到server.clients_pending_write这个队列，等待被写出
    listAddNodeHead(server.clients_pending_write,c);
}
```

```c
void beforeSleep(struct aeEventLoop *eventLoop){
    // ...
    // 定义迭代器，指向server.clients_pending_write->head;
    listIter li;
    li->next = server.clients_pending_write->head;
    li->direction = AL_START_HEAD;
    // 循环遍历待写出的client
    while ((ln = listNext(&li))) {
        // 内部调用aeApiAddEvent(fd，WRITEABLE)，监听socket的FD读事件
        // 并且绑定 写处理器 sendReplyToClient，可以把响应写到客户端socket
        connSetWriteHandlerWithBarrier(c->conn, sendReplyToClient, ae_barrier)
    }
}
```

**Redis 6.0版本中引入了多线程，目的是为了提高IO读写效率。因此在解析客户端命令、写响应结果时采用了多线程。核心的命令执行、IO多路复用模块依然是由主线程执行。**

![image-20220706131338467](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220706131338467.png)

# RESP协议2

Redis是一个CS架构的软件，通信一般分两步（不包括pipeline和PubSub）：

1. 客户端（client）向服务端（server）发送一条命令
2. 服务端解析并执行命令，返回响应结果给客户端

因此客户端发送命令的格式、服务端响应结果的格式必须有一个规范，这个规范就是通信协议。

而在Redis中采用的是RESP（Redis Serialization Protocol）协议：

* Redis 1.2版本引入了RESP协议
* Redis 2.0版本中成为与Redis服务端通信的标准，称为RESP2
* Redis 6.0版本中，从RESP2升级到了RESP3协议，增加了更多数据类型并且支持6.0的新特性--客户端缓存

但目前，默认使用的依然是RESP2协议，也是我们要学习的协议版本（以下简称RESP）。

在RESP中，通过首字节的字符来区分不同数据类型，常用的数据类型包括5种：

* 单行字符串：首字节是 ‘+’ ，后面跟上单行字符串，以CRLF（ "\r\n" ）结尾。例如返回"OK"： "+OK\r\n"

* 错误（Errors）：首字节是 ‘-’ ，与单行字符串格式一样，只是字符串是异常信息，例如："-Error message\r\n"

* 数值：首字节是 ‘:’ ，后面跟上数字格式的字符串，以CRLF结尾。例如：":10\r\n"

* 多行字符串：首字节是 ‘$’ ，表示二进制安全的字符串，最大支持512MB：

  * 如果大小为0，则代表空字符串："$0\r\n\r\n"

  * 如果大小为-1，则代表不存在："$-1\r\n"

* 数组：首字节是 ‘*’，后面跟上数组元素个数，再跟上元素，元素数据类型不限:

## 模拟redis客户端 

### socket实现

```java
public class Main {
    static Socket socket;
    static PrintWriter writer;
    static BufferedReader reader;

    public static void main(String[] args) {
        //1 建立连接
        try {
            socket = new Socket("192.168.32.3", 6379);
            //2 获取输出输入流
            writer = new PrintWriter(new OutputStreamWriter(socket.getOutputStream(), StandardCharsets.UTF_8));
            reader = new BufferedReader(new InputStreamReader(socket.getInputStream(), StandardCharsets.UTF_8));
            //3 发出请求 set name 张
            sendRequest("auth","123456");
            Object o=handlerResponse();
            System.out.println("o = " + o);
            sendRequest("set","name","张");
            //3 解析响应
            Object o1=handlerResponse();
            System.out.println("o = " + o1);
            //4 释放连接
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                if (reader != null) {
                    reader.close();
                }
                if (writer != null) writer.close();
                if (socket != null) socket.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }

    }

    private static Object handlerResponse() {
        //读取首字节
        try {
            int prefix =  reader.read();
            //判断数据类型
            switch (prefix){
                case '+': //单行字符串 直接读一行
                    return  reader.readLine();
                case '-'://异常 也读一行
                    throw  new RuntimeException(reader.readLine());
                case ':':// 数字
                    return Long.parseLong(reader.readLine());
                case '$'://读多行字符串
                    int len =Integer.parseInt(reader.readLine());
                    if (len==-1){
                        return  null;
                    }
                    if (len==0){
                        return "";
                    }
                    return  reader.readLine();
                case '*'://数组
                    return readBulkString();
                default:
                    throw new  RuntimeException("未知格式异常");
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
        return null;
    }

    private static Object readBulkString() throws IOException {
        //获取数组大小
        int len=Integer.parseInt(reader.readLine());
        if (len<=0){
            return null;
        }
        //定义集合 接受多个元素
        ArrayList<Object> list = new ArrayList<>();
        //遍历一次读取每个元素
        for (int i = 0; i < len; i++) {
          Object o=  handlerResponse();
          list.add(o);
        }
        return list;
    }

    //set name 张
    private static void sendRequest(String ... args) {
        writer.println("*"+args.length);
        for (String arg : args) {
            writer.println("$"+arg.getBytes(StandardCharsets.UTF_8).length);
            writer.println(arg);
        }
        writer.flush();
    }
}
```

### Netty实现:

```java
@Slf4j
public class TestRedis {
    public static void main(String[] args) {
        NioEventLoopGroup worker = new NioEventLoopGroup();
        byte[] LINE = {13, 10};
        try {
            Bootstrap bootstrap = new Bootstrap();
            bootstrap.channel(NioSocketChannel.class);
            bootstrap.group(worker);
            bootstrap.handler(new ChannelInitializer<SocketChannel>() {
                @Override
                protected void initChannel(SocketChannel ch) {
                    ch.pipeline().addLast(new LoggingHandler());
                    ch.pipeline().addLast(new ChannelInboundHandlerAdapter() {
                        // 会在连接 channel 建立成功后，会触发 active 事件
                        @Override
                        public void channelActive(ChannelHandlerContext ctx) {
                            auth(ctx);
                            set(ctx);
                            get(ctx);
                        }
                        private void auth(ChannelHandlerContext ctx){
                            ByteBuf buf = ctx.alloc().buffer();
                            buf.writeBytes("*2".getBytes());
                            buf.writeBytes(LINE);
                            buf.writeBytes("$4".getBytes());
                            buf.writeBytes(LINE);
                            buf.writeBytes("auth".getBytes());
                            buf.writeBytes(LINE);
                            buf.writeBytes("$6".getBytes());
                            buf.writeBytes(LINE);
                            buf.writeBytes("123456".getBytes());
                            buf.writeBytes(LINE);
                            ctx.writeAndFlush(buf);
                        }
                        private void get(ChannelHandlerContext ctx) {
                            ByteBuf buf = ctx.alloc().buffer();
                            buf.writeBytes("*2".getBytes());
                            buf.writeBytes(LINE);
                            buf.writeBytes("$3".getBytes());
                            buf.writeBytes(LINE);
                            buf.writeBytes("get".getBytes());
                            buf.writeBytes(LINE);
                            buf.writeBytes("$3".getBytes());
                            buf.writeBytes(LINE);
                            buf.writeBytes("aaa".getBytes());
                            buf.writeBytes(LINE);
                            ctx.writeAndFlush(buf);
                        }
                        private void set(ChannelHandlerContext ctx) {
                            ByteBuf buf = ctx.alloc().buffer();
                            buf.writeBytes("*3".getBytes());
                            buf.writeBytes(LINE);
                            buf.writeBytes("$3".getBytes());
                            buf.writeBytes(LINE);
                            buf.writeBytes("set".getBytes());
                            buf.writeBytes(LINE);
                            buf.writeBytes("$3".getBytes());
                            buf.writeBytes(LINE);
                            buf.writeBytes("aaa".getBytes());
                            buf.writeBytes(LINE);
                            buf.writeBytes("$3".getBytes());
                            buf.writeBytes(LINE);
                            buf.writeBytes("bbb".getBytes());
                            buf.writeBytes(LINE);
                            ctx.writeAndFlush(buf);
                        }

                        @Override
                        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
                            ByteBuf buf = (ByteBuf) msg;
                            System.out.println(buf.toString(Charset.defaultCharset()));
                        }
                    });
                }
            });
            ChannelFuture channelFuture = bootstrap.connect("192.168.32.3", 6379).sync();
            channelFuture.channel().closeFuture().sync();
        } catch (InterruptedException e) {
            log.error("client error", e);
        } finally {
            worker.shutdownGracefully();
        }
    }
}
```

# Redis内存策略

## Redis内存回收

Redis之所以性能强，最主要的原因就是基于内存存储。然而单节点的Redis其内存大小不宜过大，会影响持久化或主从同步性能。
我们可以通过修改配置文件来设置Redis的最大内存：

```sh
# 格式：
# maxmemory <bytes>
# 例如：
maxmemory 1gb
```

当内存使用达到上限时，就无法存储更多数据了。为了解决这个问题，Redis提供了一些策略实现内存回收：

* 内存过期策略
* 内存淘汰策略

在学习Redis缓存的时候我们说过，可以通过expire命令给Redis的key设置TTL（存活时间）：

![image-20220706150519388](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220706150519388.png)

可以发现，当key的TTL到期以后，再次访问name返回的是nil，说明这个key已经不存在了，对应的内存也得到释放。从而起到内存回收的目的。

这里有两个问题需要我们思考：
Redis是如何知道一个key是否过期呢？

* 利用两个Dict分别记录key-value对及key-ttl对

是不是TTL到期就立即删除了呢？

* 惰性删除
* 周期删除



Redis本身是一个典型的key-value内存存储数据库，因此所有的key、value都保存在之前学习过的Dict结构中。不过在其database结构体中，有两个Dict：一个用来记录key-value；另一个用来记录key-TTL。

```c
typedef struct redisDb {
    dict *dict;                 /* 存放所有key及value的地方，也被称为keyspace*/
    dict *expires;              /* 存放每一个key及其对应的TTL存活时间，只包含设置了TTL的key*/
    dict *blocking_keys;        /* Keys with clients waiting for data (BLPOP)*/
    dict *ready_keys;           /* Blocked keys that received a PUSH */
    dict *watched_keys;         /* WATCHED keys for MULTI/EXEC CAS */
    int id;                     /* Database ID，0~15 */
    long long avg_ttl;          /* 记录平均TTL时长 */
    unsigned long expires_cursor; /* expire检查时在dict中抽样的索引位置. */
    list *defrag_later;         /* 等待碎片整理的key列表. */
} redisDb;
```

![image-20220706150613431](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220706150613431.png)

## 惰性删除

顾明思议并不是在TTL到期后就立刻删除，而是在访问一个key的时候，检查该key的存活时间，如果已经过期才执行删除。

```c
// 查找一个key执行写操作
robj *lookupKeyWriteWithFlags(redisDb *db, robj *key, int flags) {
    // 检查key是否过期
    expireIfNeeded(db,key);
    return lookupKey(db,key,flags);
}
// 查找一个key执行读操作
robj *lookupKeyReadWithFlags(redisDb *db, robj *key, int flags) {
    robj *val;
    // 检查key是否过期    if (expireIfNeeded(db,key) == 1) {
        // ...略
    }
    return NULL;
}

int expireIfNeeded(redisDb *db, robj *key) {
    // 判断是否过期，如果未过期直接结束并返回0
    if (!keyIsExpired(db,key)) return 0;
    // ... 略
    // 删除过期key
    deleteExpiredKeyAndPropagate(db,key);
    return 1;
}
```

## 周期删除:

周期删除：顾明思议是通过一个定时任务，周期性的**抽样部分过期的key**，然后执行删除。执行周期有两种：

* Redis服务初始化函数initServer()中设置定时任务，按照server.hz的频率来执行过期key清理，模式为SLOW
* Redis的每个事件循环前会调用beforeSleep()函数，执行过期key清理，模式为FAST

```c
// server.c
void initServer(void){
    // ...
    // 创建定时器，关联回调函数serverCron，处理周期取决于server.hz，默认10
    aeCreateTimeEvent(server.el, 1, serverCron, NULL, NULL) 
}

// server.c
int serverCron(struct aeEventLoop *eventLoop, long long id, void *clientData) {
    // 更新lruclock到当前时间，为后期的LRU和LFU做准备
    unsigned int lruclock = getLRUClock();
    atomicSet(server.lruclock,lruclock);
    // 执行database的数据清理，例如过期key处理
    databasesCron();
}

void databasesCron(void) {
    // 尝试清理部分过期key，清理模式默认为SLOW
    activeExpireCycle(
          ACTIVE_EXPIRE_CYCLE_SLOW);
}

void beforeSleep(struct aeEventLoop *eventLoop){
    // ...
    // 尝试清理部分过期key，清理模式默认为FAST
    activeExpireCycle(
         ACTIVE_EXPIRE_CYCLE_FAST);
}
```

**SLOW模式规则：**

1. 执行频率受server.hz影响，默认为10，即每秒执行10次，每个执行周期100ms。
2. 执行清理耗时不超过一次执行周期的25%.默认slow模式耗时不超过25ms
3. 逐个遍历db，逐个遍历db中的bucket，抽取20个key判断是否过期
4. 如果没达到时间上限（25ms）并且过期key比例大于10%，再进行一次抽样，否则结束

**FAST模式规则（过期key比例小于10%不执行 ）：**

1. 执行频率受beforeSleep()调用频率影响，但两次FAST模式间隔不低于2ms
2. 执行清理耗时不超过1ms
3. 逐个遍历db，逐个遍历db中的bucket，抽取20个key判断是否过期
4. 如果没达到时间上限（1ms）并且过期key比例大于10%，再进行一次抽样，否则结束



RedisKey的TTL记录方式：

* 在RedisDB中通过一个Dict记录每个Key的TTL时间

过期key的删除策略：

* 惰性清理：每次查找key时判断是否过期，如果过期则删除
* 定期清理：定期抽样部分key，判断是否过期，如果过期则删除。

定期清理的两种模式：

* SLOW模式执行频率默认为10，每次不超过25ms
* FAST模式执行频率不固定，但两次间隔不低于2ms，每次耗时不超过1ms

# 内存淘汰

内存淘汰：就是当Redis内存使用达到设置的上限时，主动挑选部分key删除以释放更多内存的流程。Redis会在处理客户端命令的方法processCommand()中尝试做内存淘汰：

```c
int processCommand(client *c) {
    // 如果服务器设置了server.maxmemory属性，并且并未有执行lua脚本
    if (server.maxmemory && !server.lua_timedout) {
        // 尝试进行内存淘汰performEvictions
        int out_of_memory = (performEvictions() == EVICT_FAIL);
        // ...
        if (out_of_memory && reject_cmd_on_oom) {
            rejectCommand(c, shared.oomerr);
            return C_OK;
        }
        // ....
    }
}
```

Redis支持8种不同策略来选择要删除的key：

* noeviction： 不淘汰任何key，但是内存满时不允许写入新数据，默认就是这种策略。
* volatile-ttl： 对设置了TTL的key，比较key的剩余TTL值，TTL越小越先被淘汰
* allkeys-random：对全体key ，随机进行淘汰。也就是直接从db->dict中随机挑选
* volatile-random：对设置了TTL的key ，随机进行淘汰。也就是从db->expires中随机挑选。
* allkeys-lru： 对全体key，基于LRU算法进行淘汰
* volatile-lru： 对设置了TTL的key，基于LRU算法进行淘汰
* allkeys-lfu： 对全体key，基于LFU算法进行淘汰
* volatile-lfu： 对设置了TTL的key，基于LFI算法进行淘汰

比较容易混淆的有两个：

* LRU（Least Recently Used），最少最近使用。用当前时间减去最后一次访问时间，这个值越大则淘汰优先级越高。
* LFU（Least Frequently Used），最少频率使用。会统计每个key的访问频率，值越小淘汰优先级越高。

Redis的数据都会被封装为RedisObject结构：

```c
typedef struct redisObject {
    unsigned type:4;        // 对象类型
    unsigned encoding:4;    // 编码方式
    unsigned lru:LRU_BITS;  // LRU：以秒为单位记录最近一次访问时间，长度24bit
			  // LFU：高16位以分钟为单位记录最近一次访问时间，低8位记录逻辑访问次数
    int refcount;           // 引用计数，计数为0则可以回收
    void *ptr;              // 数据指针，指向真实数据
} robj;

```

LFU的访问次数之所以叫做逻辑访问次数，是因为并不是每次key被访问都计数，而是通过运算：

* 生成0~1之间的随机数R
* 计算 (旧次数 * lfu_log_factor + 1)，记录为P
* 如果 R < P ，则计数器 + 1，且最大不超过255
* 访问次数会随时间衰减，距离上一次访问时间每隔 lfu_decay_time 分钟，计数器 -1

![image-20220706160511015](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220706160511015.png)

当eviction_pool池子满了后，DB中的元素要比较当前元素的idleTIme是否比eviction_pool中最小的还要大，否则就淘汰；
