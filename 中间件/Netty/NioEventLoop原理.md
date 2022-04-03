事件调度层：EventLoop 
----------------------

**EventLoop 可以说是 Netty 的调度中心，负责监听多种事件类型：I/O 事件、信号事件、定时事件**

线程模型：单线程，多线程，主从多线程

 Reactor 线程模型运行机制的四个步骤，分别为连接注册、事件轮询、事件分发、任务处理，如下图所示。



<img src="assets/CgqCHl-ZNEGAMU-zAAEsYdWKArA085.png" alt="4.png" style="zoom:67%;" />

连接注册：Channel 建立后，注册至 Reactor 线程中的 Selector 选择器。

事件轮询：轮询 Selector 选择器中已注册的所有 Channel 的 I/O 事件。

事件分发：为准备就绪的 I/O 事件分配相应的处理线程。

任务处理：Reactor 线程还负责任务队列中的非 I/O 任务，每个 Worker 线程从各自维护的任务队列中取出任务异步执行。

EventLoop 是一种<u>事件等待和处理的程序模型</u>，可以解决多线程资源消耗高的问题。例如 Node.js 就采用了 EventLoop 的运行机制，不仅占用资源低，而且能够支撑了大规模的流量访问。

下图展示了 EventLoop 通用的运行模式。每当事件发生时，应用程序都会将产生的事件放入事件队列当中，然后 EventLoop 会轮询从队列中取出事件执行或者将事件分发给相应的事件监听者执行。事件执行的方式通常分为立即执行、延后执行、定期执行几种。

<img src="assets/Ciqc1F-ZNFKAAZr4AANvWWMqnKw586.png" alt="5.png" style="zoom:67%;" />

EventLoop 可以理解为 Reactor 线程模型的事件处理引擎，每个 EventLoop 线程都维护一个 Selector 选择器和任务队列 taskQueue。它主要负责处理 I/O 事件、普通任务和定时任务。 NioEventLoop是Netty最推荐使用的一个类

```java
//不断轮询去接受请求和任务
protected void run() {
    for (;;) {
        try {
            try {
                switch (selectStrategy.calculateStrategy(selectNowSupplier, hasTasks())) {
                case SelectStrategy.CONTINUE:
                    continue;
                case SelectStrategy.BUSY_WAIT:
                case SelectStrategy.SELECT:
                    select(wakenUp.getAndSet(false)); // 轮询 I/O 事件
                    if (wakenUp.get()) {
                        selector.wakeup();
                    }
                default:
                }
            } catch (IOException e) {
                rebuildSelector0();
                handleLoopException(e);
                continue;
            }

            cancelledKeys = 0;
            needsToSelectAgain = false;
            final int ioRatio = this.ioRatio;
            if (ioRatio == 100) {//调整 I/O 事件处理和任务处理的时间比例
                try {
                    processSelectedKeys(); // 处理 I/O 事件
                } finally {
                    runAllTasks(); // 处理所有任务
                }
            } else {
                final long ioStartTime = System.nanoTime();
                try {
                    processSelectedKeys(); // 处理 I/O 事件
                } finally {
                    final long ioTime = System.nanoTime() - ioStartTime;
                    runAllTasks(ioTime * (100 - ioRatio) / ioRatio); // 处理完 I/O 事件，再处理异步任务队列
                }
            }
        } catch (Throwable t) {
            handleLoopException(t);
        }
        try {
            if (isShuttingDown()) {
                closeAll();
                if (confirmShutdown()) {
                    return;
                }
            }
        } catch (Throwable t) {
            handleLoopException(t);
        }
    }
}
```

NioEventLoop 的事件处理机制采用的是**无锁串行化**的设计思路。

缺点：不能执行时间过长的 I/O 操作，一旦某个 I/O 事件发生阻塞，那么后续的所有 I/O 事件都无法执行，甚至造成事件积压

##### 如何解决JDK epoll 空轮询的 Bug

```java
long time = System.nanoTime();
if (time - TimeUnit.MILLISECONDS.toNanos(timeoutMillis) >= currentTimeNanos) {
    selectCnt = 1;
} else if (SELECTOR_AUTO_REBUILD_THRESHOLD > 0 &&
        selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) {
    selector = selectRebuildSelector(selectCnt);
    selectCnt = 1;
    break;
}
```

Netty 提供了一种检测机制判断线程是否可能陷入空轮询，具体的实现方式如下：

1.   每次执行 Select 操作之前记录当前时间 currentTimeNanos。

2.   time - TimeUnit.MILLISECONDS.toNanos(timeoutMillis) >= currentTimeNanos，如果事**件轮询的持续时间大于等于 timeoutMillis**，那么说明是正常的，否则表明阻塞时间并未达到预期，可能触发了空轮询的 Bug。

3.   Netty 引入了**计数变量 selectCnt**。在正常情况下，selectCnt 会重置，否则会对 selectCnt 自增计数。当 selectCnt 达到SELECTOR_AUTO_REBUILD_THRESHOLD（默认512） 阈值时，会触发重建 Selector 对象。

Netty 采用这种方法巧妙地规避了 JDK Bug。异常的 Selector 中所有的 SelectionKey 会重新注册到新建的 Selector 上，重建完成之后异常的 Selector 就可以废弃了。

##### 任务处理机制

1.   fetchFromScheduledTaskQueue 函数：将定时任务从 scheduledTaskQueue 中取出，聚合放入普通任务队列 taskQueue 中，只有定时任务的截止时间小于当前时间才可以被合并。
2.   从普通任务队列 taskQueue 中取出任务。
3.   计算任务执行的最大超时时间。
4.   safeExecute 函数：安全执行任务，实际直接调用的 Runnable 的 run() 方法。
5.   <u>每执行 64 个任务进行超时时间的检查，如果执行时间大于最大超时时间，则立即停止执行任务，避免影响下一轮的 I/O 事件的处理。</u>

6.   最后获取尾部队列中的任务执行。



##### EventLoop 最佳实践

网络连接建立过程中三次握手、安全认证的过程会消耗不少时间。这里建议采用 **Boss 和 Worker 两个 EventLoopGroup**，有助于分担 Reactor 线程的压力。

由于 Reactor 线程模式**适合处理耗时短的任务场景**，对于耗时较长的 ChannelHandler 可以考虑维护一个业务线程池，将编解码后的数据封装成 Task 进行异步处理，避免 ChannelHandler 阻塞而造成 EventLoop 不可用。

如果业务逻辑执行时间较短，建议直接在 ChannelHandler 中执行。例如编解码操作，这样可以避免过度设计而造成架构的复杂性。

**不宜设计过多的 ChannelHandler**。对于系统性能和可维护性都会存在问题，在设计业务架构的时候，需要明确业务分层和 Netty 分层之间的界限。不要一味地将业务逻辑都添加到 ChannelHandler 中。