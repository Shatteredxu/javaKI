服务编排层：Pipeline +Handler 
------------------------------

ChannelPipeline 和 ChannelHandler

<img src="assets/CgqCHl-dLiiAcORMAAYJnrq5ceE455.png" alt="image.png" style="zoom:40%;" />

每个 Channel 会绑定一个 ChannelPipeline，每一个 ChannelPipeline 都包含多个 ChannelHandlerContext，所有 ChannelHandlerContext 之间组成了双向链表

**ChannelHandlerContext** 用于保存 ChannelHandler 上下文；ChannelHandlerContext 则包含了 ChannelHandler 生命周期的所有事件，如 connect、bind、read、flush、write、close 等

ChannelPipeline 的双向链表分别维护了 **HeadContext 和 TailContext 的头尾节点**，

