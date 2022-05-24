BootStrap & ServerBootStrap启动过程
-----------------------------------

网络通信层主要是用来接受连接事件，当网络数据读取到内核缓冲区后（放到内核缓存区之前是操作系统epoll干的事情，<u>网络通信层负责监听</u>），会触发各种网络事件，这些网络事件会分发给事件调度层进行处理

以下就是一个HTTP 服务器构建过程，

```java
public class HttpServer {
    public void start(int port) throws Exception {
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .localAddress(new InetSocketAddress(port))
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        public void initChannel(SocketChannel ch) {
                            ch.pipeline()
                                    .addLast("codec", new HttpServerCodec())                  // HTTP 编解码
                                    .addLast("compressor", new HttpContentCompressor())       // HttpContent 压缩
                                    .addLast("aggregator", new HttpObjectAggregator(65536))   // HTTP 消息聚合
                                    .addLast("handler", new HttpServerHandler());             // 自定义业务逻辑处理器
                        }
                    })
                    .childOption(ChannelOption.SO_KEEPALIVE, true);
            ChannelFuture f = b.bind().sync();
            System.out.println("Http Server started， Listening on " + port);
            f.channel().closeFuture().sync();
        } finally {
            workerGroup.shutdownGracefully();
            bossGroup.shutdownGracefully();
        }
    }
    public static void main(String[] args) throws Exception {
        new HttpServer().start(8088);
    }
}
```

Netty 服务端的启动过程大致分为三个步骤：

1.   配置线程池；(两个线程池，分别为bossGroup,workerGroup,主从多线程模式)

2.   Channel 初始化；

     >   1.   Netty 服务端采用 NioServerSocketChannel 作为 Channel 的类型，客户端采用 NioSocketChannel
     >   2.   通过 ChannelPipeline 去注册多个 ChannelHandler
     >   3.   设置 Channel 参数，`SO_KEEPALIVE`连接保活

3.   端口绑定。

     >   ChannelFuture f = b.bind().sync();sync() 方法则会阻塞，直至整个启动过程完成