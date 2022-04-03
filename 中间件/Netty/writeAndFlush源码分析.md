writeAndFlush 主要分为两个步骤，write 和 flush。通过上面的分析可以看出只调用 write 方法，数据并不会被真正发送出去，而是存储在 ChannelOutboundBuffer 的缓存内

ChannelOutboundBuffer :是一个链表结构，每次传入的数据都会被封装成一个 Entry 对象添加到链表中。ChannelOutboundBuffer 包含**三个非常重要的指针**：第一个被写到缓冲区的**节点 flushedEntry**、第一个未被写到缓冲区的**节点 unflushedEntry**和最后一个**节点 tailEntry。**