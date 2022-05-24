## Netty服务端启动过程

在使用Netty作为服务器端时，主要分为几个步骤：

- 配置 EventLoopGroup 线程组；（bossGroup，workerGroup ）
- 配置 Channel 的类型；（NioServerSocketChannel）
- 设置 ServerSocketChannel 对应的 Handler；
- 设置网络监听的端口（bind）；
- 设置 SocketChannel 对应的 Handler；
- 配置 Channel 参数。

##### 首先从Bind开始

调用doBind绑定地址，初始化channel，最后调用 operationComplete()在通过` doBind0(regFuture, channel, localAddress, promise);`绑定

```java
public ChannelFuture bind() {
    validate();
    SocketAddress localAddress = this.localAddress;
    if (localAddress == null) {
        throw new IllegalStateException("localAddress not set");
    }
    return doBind(localAddress);
}

private ChannelFuture doBind(final SocketAddress localAddress) {
    final ChannelFuture regFuture = initAndRegister();//初始化并注册 Channel，同时返回一个 ChannelFuture 实例 regFuture
    final Channel channel = regFuture.channel();
    if (regFuture.cause() != null) {//查看有没有出现异常
        return regFuture;
    }
·/**
 initAndRegister() 是否执行完毕，如果执行完毕则调用 doBind0() 进行 Socket 绑定。
 如果 initAndRegister() 还没有执行结束，regFuture 会添加一个 ChannelFutureListener 回调监听，
 当 initAndRegister() 执行结束后会调用 operationComplete()，同样通过 doBind0() 进行端口绑定。
  **/
    if (regFuture.isDone()) {//

        ChannelPromise promise = channel.newPromise();
        doBind0(regFuture, channel, localAddress, promise);
        return promise;
    } else {
        final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
        regFuture.addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                Throwable cause = future.cause();
                if (cause != null) {
                    promise.setFailure(cause);
                } else {
                    promise.registered();
                    doBind0(regFuture, channel, localAddress, promise);
              }
            }
        });
        return promise;
    }

}
```

##### 再看看initAndRegister方法

初始化，注册channel

##### Pipeline 事件传播

完成 Channel 向 Selector 注册后，接下来就会触发 Pipeline 一系列的事件传播。在事件传播之前，用户自定义的业务处理器是如何被添加到 Pipeline 中的呢？答案就在`pipeline.invokeHandlerAddedIfNeeded()` 当中，我们重点看下 handlerAdded 事件的处理过程