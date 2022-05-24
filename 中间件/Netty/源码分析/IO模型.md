### IO模型

当我们一个线程发送消息后，线程是否等待消息返回分为**阻塞和非阻塞**，我们选择通过轮询方式查询数据还是操作系统通知我们数据准备就绪分为**同步和异步**，IO模型主要有这几种：同步阻塞，同步非阻塞，异步阻塞，异步非阻塞，IO多路复用（select/poll/epoll）

https://www.bilibili.com/video/BV1qJ411w7du/?spm_id_from=333.788.recommend_more_video.-1

https://blog.csdn.net/XueyinGuo/article/details/113096163

https://zhuanlan.zhihu.com/p/115912936

**流**：将数据抽象成为一组有顺序的字节集合，主要为数据建立一个输送管道，将数据从输入端传输到输出端

**五种 IO 模型包括**：阻塞 IO、非阻塞 IO、信号驱动 IO、IO 多路复用、异步 IO。其中，前四个被称为同步 IO；

**同步和异步说的是消息的通知机制，阻塞非阻塞说的是线程的状态**

##### 阻塞 IO（Blocking I/O）

*一直等着数据到来，不去做其他的事情*

BIO 中的阻塞，就是阻塞在 2 个地方：

1. OS 等待数据报(通过网络发送过来)准备好。
2. 将数据从内核空间拷贝到用户空间。

##### 非阻塞 IO（Noblocking I/O）

![avatar](assets/非阻塞IO模型.png)

每次应用进程询问内核是否有数据报准备好，当有数据报准备好时，就进行拷贝数据报的操作，***从内核拷贝到用户空间，和拷贝完成返回的这段时间，应用进程是阻塞的***。但在没有数据报准备好时，并不会阻塞程序，内核直接返回未准备就绪的信号，等待应用进程的下一个轮询。但是，轮询对于 CPU 来说是较大的浪费，一般只有在**特定的场景**下才使用。

**NIO 不会在 recvfrom（询问数据是否准备好）时阻塞，但还是会在将数据从内核空间拷贝到用户空间时阻塞。一定要注意这个地方，Non-Blocking 还是会阻塞的。**

##### IO 多路复用（I/O Multiplexing）

https://zhuanlan.zhihu.com/p/115220699


传统情况下 client 与 server 通信需要 3 个 socket(客户端的 socket，服务端的 server socket，服务端中用来和客户端通信的 socket)，而在 IO 多路复用中，客户端与服务端通信需要的不是 socket，而是 3 个 channel，通过 channel 可以完成与 socket 同样的操作，channel 的底层还是使用的 socket 进行通信，但是多个 channel 只对应一个 socket(可能不只是一个，但是 socket 的数量一定少于 channel 数量)，这样仅仅通过少量的 socket 就可以完成更多的连接，提高了 client 容量。

IO多路复用就是在单线程/进程中处理多个事件流，更少的浪费资源，在不同的操作系统中有不同的实现：

1. Windows：selector
2. Linux：select、poll、epoll
3. Mac：kqueue

其中 epoll，kqueue 比 selector 更为高效，这是因为他们监听方式的不同。selector 的监听是通过轮询 FD_SETSIZE 来问每一个 socket：“你改变了吗？”，假若监听到事件，那么 selector 就会调用相应的事件处理器进行处理。但是 epoll 与 kqueue 不同，他们把 socket 与事件绑定在一起，当监听到 socket 变化时，立即可以调用相应的处理。 **selector，epoll，kqueue 都属于 Reactor IO 设计。**

##### 信号驱动（Signal driven IO）

![avatar](assets/信号驱动IO模型.png)

信号驱动 IO 模型，应用进程告诉内核：当数据报准备好的时候，给我发送一个信号，对 SIGIO 信号进行捕捉，并且调用我的信号处理函数来获取数据报。

##### 异步 IO（Asynchronous I/O）

程序发起 read 操作之后，立刻就可以开始去做其它的事。kernel 会等待数据准备完成，然后将数据拷贝到用户内存，当这一切都完成之后，kernel 会给用户进程发送一个 signal，告诉它 read 操作完成了。

##### TODO 什么是Reactor IO ？

##### Blocking IO 与 Non-Blocking IO 区别？

阻塞或非阻塞只涉及程序和 OS，Blocking IO 会一直 block 程序直到 OS 返回，而 Non-Block IO 在 OS 内核准备数据包的情况下会立即得到返回。

##### Asynchronous IO 与 Synchronous IO？

只要有 block 就是同步 IO，完全没有 block 则是异步 IO。所以我们之前所说的 Blocking IO、Non-Blocking IO、IO Multiplex，均为 Synchronous IO，只有 Asynchronous IO 为异步 IO。

#####  Non-Blocking IO 不是会立即返回没有阻塞吗?

**Non-Blocking IO 在数据包准备时是非阻塞的，但是在将数据从内核空间拷贝到用户空间还是会阻塞**。而 Asynchronous IO 则不一样，当进程发起 IO 操作之后，就直接返回再也不理睬了，由内核完成读写，完成读写操作后 kernel 发送一个信号，告诉进程说 IO 完成。在这整个过程中，进程完全没有被 block。

##### IO 模式（Reactor 与 Proactor）

##### 同步与异步，阻塞与非阻塞他們的区别

##### Java 中的 NIO

https://www.cnblogs.com/loveer/p/11479887.html

NIO 是一种基于通道和缓冲区的 I/O 方式，它可以使用 Native 函数库直接分配堆外内存（区别于 JVM 的运行时数据区），然后通过一个存储在 java 堆里面的 DirectByteBuffer 对象作为这块内存的直接引用进行操作。这样能在一些场景显著提高性能，因为避免了在 Java 堆和 Native 堆中来回复制数据。

NIO 主要有三大核心部分：Channel(通道)，Buffer(缓冲区), Selector（选择器）。*传统 IO 是基于字节流和字符流进行操作*（基于流），而 NIO 基于 Channel 和 Buffer(缓冲区)进行操作，**数据总是从通道读取到缓冲区中，或者从缓冲区写入到通道中。Selector(选择区)用于监听多个通道的事件（比如：连接打开，数据到达）**。因此，单个线程可以监听多个数据通道。

Channel: 可以通过它读取和写入数据。可以把它看做是IO中的流,buffer负责存储数据，channel负责传输数据

Selector（选择器）:用于监听多个通道的事件（比如：连接打开，数据到达）。因此，单个的线程可以监听多个数据通道。即用选择器，借助单一线程，就可对数量庞大的活动 I/O 通道实施监控和维护。

服务器上所有 Channel 需要向 Selector 注册，而 Selector 则负责监视这些 Socket 的 IO 状态(观察者)，当其中任意一个或者多个 Channel 具有可用的 IO 操作时，该 Selector 的 select()方法将会返回大于 0 的整数，该整数值就表示该 Selector 上有多少个 Channel 具有可用的 IO 操作，并提供了 selectedKeys（）方法来返回这些 Channel 对应的 SelectionKey 集合(一个 SelectionKey 对应一个就绪的通道)。正是通过 Selector，使得服务器端只需要不断地调用 Selector 实例的 select()方法即可知道当前所有 Channel 是否有需要处理的 IO 操作。注：**java NIO 就是多路复用 IO，jdk7 之后底层是 epoll 模型**。

##### Java 中的 AIO

JDK1.7 升级了 NIO 类库，升级后的 NIO 类库被称为 NIO 2.0。Java 正式提供了异步文件 I/O 操作，同时提供了与 UNIX 网络编程事件驱动 I/O 对应的 AIO。**NIO 2.0 引入了新的异步通道的概念，并提供了异步文件通道和异步套接字通道的实现**。

##### epoll和select的区别

[参考链接](https://github.com/doocs/source-code-hunter/blob/main/docs/Netty/IOTechnologyBase/%E6%8A%8A%E8%A2%AB%E8%AF%B4%E7%83%82%E7%9A%84BIO%E3%80%81NIO%E3%80%81AIO%E5%86%8D%E4%BB%8E%E5%A4%B4%E5%88%B0%E5%B0%BE%E6%89%AF%E4%B8%80%E9%81%8D.md)

#### IO多路复用（select/poll/epoll）

##### select

1.   生成文件描述符，也就是创建socket连接
2.   找到最大的文件描述符
3.   调用select(max+1,&rset(一个bitmap))，select是一个系统调用，会将rset数组全量拷贝到内核态，如果没有数据则会一直阻塞，如果有数据则系统会将bitmap进行置位，并返回。
4.   max+1的作为是会将0-max+1的文件描述符拿出来，而不会全部拿到
5.   最后返回后的set进行遍历,取出为1，已经准备好数据的socket。系统调用完成后，**返回可读可写的数量之和**

缺点：1.默认1024,有上限 2.rset被内核修改后，需要重新遍历改回原来的值3.拷贝到内核有开销4.O(n)遍历时间

优势：通过一次系统调用把所有的fds传递给内核，内核进行遍历，这种遍历减少了BIO的多次系统调用的开销

##### poll

1.   放弃bitmap，采用结构体pollfd{int fd;short events;short revents}
2.   同样是阻塞函数，放到内核去置位
3.   置位是pollfd的revents字段(0->1)
4.   数组，不限制大小；采用结构体，恢复字段即可

优势：1没有了1024的限制，2.为文件描述符创建的结构体中有revents字段，用每次重新构造类似于readset那样的bitmap

##### epoll

相比前两个epoll不需要每次都构建传递一个巨大的文件描述符数组并将其传递到内核中，而是简单地获取一个`epoll`实例并将文件描述符**注册到它上面**

1.   首先epoll_create 在内核开辟一块内存空间，就是一个结构体，**结构体为（红黑树+链表）**
2.   每当有客户端连接，网卡就有数据到达，产生硬件中断，调用epoll_add返回文件描述符放到红黑树中
3.   网卡有数据到达，就会将网卡buffer中的数据拷贝到对应客户端的文件描述符的内存区域，并且将文件描述符放到epoll_create创建的链表中，
4.   应用程序调用epoll_wait()直接获取有事件的文件描述符进行读写。

<img src="assets/epoll工作示意图.png" alt="在这里插入图片描述" style="zoom:47%;" />

###### 工作模式（LT/ET）

**LT模式：（水平触发）**

​      fd可读之后，如果服务程序读走一部分就结束此次读取，LT模式下该文件描述符仍然可读
​              fd可写之后，如果服务程序写了一部分就结束此次写入，LT模式下该文件描述符也仍然可写
​       **ET模式(边缘触发)**

*   当文件描述符关联的读内核缓冲区由空转化为非空的时候，则发出可读信号进行通知，
*   当文件描述符关联的内核写缓冲区由满转化为不满的时候，则发出可写信号进行通知，



