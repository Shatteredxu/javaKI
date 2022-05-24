GO中的epoll源码解析
-------------------

[结合epoll底层看](../..\中间件\Netty\源码分析\IO模型.md)

官方的网络监听案例：

```go
func main() {
 //构造一个listener
 listener, _ := net.Listen("tcp", "127.0.0.1:9008")
 for {
  //接收请求
  conn, err := listener. ()
  //启动一个协程来处理
  go process(conn)
 }
}

func process(conn net.Conn) {
 //结束时关闭连接
 defer conn.Close()
 //读取连接上的数据
 var buf [1024]byte
 len, err := conn.Read(buf[:])
 //发送数据
 _, err = conn.Write([]byte("I am server!"))
 ...
}
```

在其他語言中，我们使用listener.Accept()来接受结果，他们会将当前线程阻塞掉，如果请求到来，则返回，这样的代码上下文切换十分好事，所以性能也非常差，但是这样的代码在Go中运行性能却是非常的不错，为啥呢？这就得学习Golang底层是怎么做的。

### **Listen 底层过程**

在 golang net 的 listen 中，会完成如下几件事：

*   创建 socket 并设置非阻塞，

    >   创建socket是一个系统调用， syscall.SetNonblock 将其设置为非阻塞模式

*   bind 绑定并监听本地的一个端口，调用 listen 开始监听

*   epoll_create 创建一个 epoll 对象

*   epoll_etl 将 listen 的 socket 添加到 epoll 中等待连接到来

### Accept 过程

服务端在 Listen 完了之后，就是对 Accept 的调用了。该函数主要做了三件事

*   调用 accept 系统调用接收一个连接
*   如果没有连接到达，把当前协程阻塞掉（gopark阻塞）
*   新连接到来的话，将其添加到 epoll 中管理，然后返回

### Read 和 Write 内部过程

1.   Read 函数会进入到 FD 的 Read 中。在这个函数内部调用 Read 系统调用来读取数据。如果数据还尚未到达则也是把自己阻塞起来
2.   调用 Write 系统调用发送数据，如果内核发送缓存区不足的时候，就把自己先阻塞起来，然后等可写时间发生的时候再继续发送

### Golang 唤醒

当要等待的事件就绪的时候，被阻塞掉的协程又是如何被重新调度的呢？Go 语言的运行时会在调度或者系统监控中调用 sysmon，它会调用 netpoll(0)，来不断地调用 epoll_wait 来查看 epoll 对象所管理的文件描述符中哪一个有事件就绪需要被处理了。如果有，就唤醒对应的协程来进行执行。在 epoll 返回的时候，ev.data 中是就绪的网络 socket 的文件描述符。根据网络就绪 fd 拿到 pollDesc。在 netpollready 中，将对应的协程推入可运行队列等待调度执行。