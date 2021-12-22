Netty 数据传输载体 ByteBuf 
---------------------------

JDK NIO 的 ByteBuffer存在很多问题：

第一，ByteBuffer 分配的长度是固定的，无法动态扩缩容，所以很难控制需要分配多大的容量。如果分配太大容量，容易造成内存浪费；如果分配太小，存放太大的数据会抛出 BufferOverflowException 异常。在使用 ByteBuffer 时，为了避免容量不足问题，你必须每次在存放数据的时候对容量大小做校验，如果超出 ByteBuffer 最大容量，那么需要重新开辟一个更大容量的 ByteBuffer，将已有的数据迁移过去。整个过程相对烦琐，对开发者而言是非常不友好的。

第二，ByteBuffer 只能通过 position 获取当前可操作的位置，因为读写共用的 position 指针，所以需要频繁调用 flip、rewind 方法切换读写状态，开发者必须很小心处理 ByteBuffer 的数据读写，稍不留意就会出错。

#### ByteBuf 结构

主要由：**读指针 readerIndex**、**写指针 writeIndex**、**最大容量 maxCapacity**

#### ByteBuf 分类

可以划分为三个不同的维度：**Heap/Direct**、**Pooled/Unpooled**和**Unsafe/非 Unsafe**

![image](assets/ByteBuf分类.png)

**Heap/Direct 就是堆内和堆外内存**。Heap 指的是在 JVM 堆内分配，底层依赖的是字节数据；Direct 则是堆外内存，不受 JVM 限制，分配方式依赖 JDK 底层的 directByteBuffer。

**Pooled/Unpooled 表示池化还是非池化内存**。Pooled 是从预先分配好的内存中取出，使用完可以放回 ByteBuf 内存池，等待下一次分配。而 Unpooled 是直接调用系统 API 去申请内存，确保能够被 JVM GC 管理回收。

**Unsafe/非 Unsafe 的区别在于操作方式是否安全。** Unsafe 表示每次调用 JDK 的 Unsafe 对象操作物理内存，依赖 offset + index 的方式操作数据。非 Unsafe 则不需要依赖 JDK 的 Unsafe 对象，直接通过数组下标的方式操作数据。

#### 内存分配器byteBufAllocator

分配内存

实现类：unpooledByteBufAllocator,pooledByteBufAllocator,

##### unpooledByteBufAllocator

heap内存分配

direct内存分配

##### pooledByteBufAllocator

参考;https://www.cnblogs.com/xiangnan6122/p/10205431.html

PooledByteBufAllocator是通过自己取一块连续的内存进行ByteBuf的封装，同样也重写了AbstractByteBuf的<u>newDirectBuffer</u>和<u>newHeapBuffer</u>两个抽象方法

