Netty堆外内存
-------------

##### 为什么使用堆外内存？

1.   堆内内存由 JVM GC 自动回收内存，降低了 Java 用户的使用心智，但是 GC 是需要时间开销成本的，堆外内存由于不受 JVM 管理，所以在一定程度上可以**降低 GC 对应用运行时带来的影响**。

2.   堆外内存需要手动释放，这一点跟 C/C++ 很像，稍有不慎就会造成应用程序**内存泄漏**，当出现内存泄漏问题时排查起来会相对困难。

3.   当进行网络 I/O 操作、文件读写时，堆内内存都需要转换为堆外内存，然后再与底层设备进行交互，所以直接使用堆外内存可以**减少一次内存拷贝**。

4.   堆外内存可以实现进程之间、JVM 多实例之间的**数据共享**。

##### 堆外内存的分配

两种方式：ByteBuffer#allocateDirect和Unsafe#allocateMemory

###### ByteBuffer#allocateDirect

```java
    DirectByteBuffer(int cap) {           // package-private
        super(-1, 0, cap, cap);
        boolean pa = VM.isDirectMemoryPageAligned();
        int ps = Bits.pageSize();
        long size = Math.max(1L, (long)cap + (pa ? ps : 0));
        Bits.reserveMemory(size, cap);

        long base = 0;
        try {
            base = unsafe.allocateMemory(size);//真正分配堆外内存，还是通过unsafe
        } catch (OutOfMemoryError x) {
            Bits.unreserveMemory(size, cap);
            throw x;
        }
        unsafe.setMemory(base, size, (byte) 0);
        if (pa && (base % ps != 0)) {
            // Round up to page boundary
            address = base + ps - (base & (ps - 1));
        } else {
            address = base;
        }
        cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
        att = null;
    }
```

##### Unsafe#allocateMemory



allocateMemory是native方法

##### 堆外内存的回收

堆外内存的回收如果不及时，很有可能造成机器的物理内存耗尽，所以我们可以通过directByteBuffer 在初始化时会创建的 Cleaner 对象，它会负责堆外内存的回收工作。那么 <u>Cleaner 是如何与 GC 关联起来的呢？</u>

java对象的四种引用：强软弱虚应用，而**Cleaner 就属于 PhantomReference 的子类**，

初始化堆外内存时，内存中的对象引用情况如下图所示，first 是 Cleaner 类中的静态变量，Cleaner 对象在初始化时会加入 Cleaner 链表中。DirectByteBuffer 对象包含堆外内存的地址、大小以及 Cleaner 对象的引用，<u>ReferenceQueue 用于保存需要回收的 Cleaner 对象</u>

当发生GC时，DirectByteBuffer 对象被回收，Cleaner 对象不再有任何引用关系，在下一次 GC 时，该 Cleaner 对象将被添加到 ReferenceQueue 中，并执行 clean() 方法。clean() 方法主要做两件事情：**1.将 Cleaner 对象从 Cleaner 链表中移除；2.调用 unsafe.freeMemory 方法清理堆外内存。**

<img src="assets/Ciqc1F-063GAc4TOAATJbR2Lmao239.png" alt="图片3.png" style="zoom: 50%;" />

### Netty的数据传输载体 ByteBuf

JDK包中的NIO提供了ByteBuffer 类，但是这个类有一些缺点：

1.   ByteBuffer 分配的长度是固定的，无法动态扩缩容
2.   ByteBuffer 只能通过 position 获取当前可操作的位置，因为读写共用的 position 指针，所以需要频繁调用 flip、rewind 方法切换读写状态，对开发人员要求较高，

所以Netty自己实现了一个高性能的数据载体ByteBuf类，可以实现：<u>容量可以按需动态扩展</u>，类似于 StringBuffer；<u>读写采用了不同的指针</u>，读写模式可以随意切换，<u>通过内置的复合缓冲类型可以实现零拷贝</u>；<u>支持引用计数</u>；<u>支持缓存池</u>。

##### Bytebuf结构

<img src="assets/CgqCHl-3uraAAhvwAASZGuNRMtA960.png" alt="Netty11（2）.png" style="zoom:50%;" />

ByteBuf 包含三个指针：读指针 readerIndex、写指针 writeIndex、最大容量 maxCapacity

##### 引用计数

ByteBuf 是基于引用计数设计的，它实现了 ReferenceCounted 接口，ByteBuf 的生命周期是由引用计数所管理。只要引用计数大于 0，表示 ByteBuf 还在被使用，如果引用计数为0，则ByteBuf就会被放进缓存池中，避免每次都重复创建

而引用计数可以成为java检测内存泄漏的工具，Netty 会对分配的 ByteBuf 进行抽样分析，检测 ByteBuf 是否已经不可达且引用计数大于 0，判定内存泄漏的位置并输出到日志中，你需要关注日志中 <u>LEAK 关键字</u>。

##### ByteBuf 分类

有三类：Heap/Direct、Pooled/Unpooled和Unsafe/非 Unsafe

<img src="assets/Ciqc1F-3h3WAMF4CAAe4IOav4SA876.png" alt="image (3).png" style="zoom: 43%;" />

Heap/Direct 就是**堆内和堆外内存**。Heap 指的是在 JVM 堆内分配，底层依赖的是字节数据；Direct 则是堆外内存，不受 JVM 限制，分配方式依赖 JDK 底层的 ByteBuffer。

Pooled/Unpooled 表示**池化还是非池化内存**。Pooled 是从预先分配好的内存中取出，使用完可以放回 ByteBuf 内存池，等待下一次分配。而 Unpooled 是直接调用系统 API 去申请内存，确保能够被 JVM GC 管理回收。

Unsafe/非 Unsafe 的区别在于**操作方式是否安全**。 Unsafe 表示每次调用 JDK 的 Unsafe 对象操作物理内存，依赖 offset + index 的方式操作数据。非 Unsafe 则不需要依赖 JDK 的 Unsafe 对象，直接通过数组下标的方式操作数据。

Netty内存分配和管理
-------------------

Netty内存分配和管理有参考jemalloc （ 在 FreeBSD 项目中引入的新一代内存分配器），jemalloc 内存分配主要有两个作用：

* 高效的内存分配和回收，提升单线程或者多线程场景下的性能。
* 减少内存碎片，包括**内部碎片和外部碎片**，提高内存的有效利用率。

#### 常用内存分配器算法

##### 动态内存分配

##### 伙伴算法

伙伴算法是一种非常经典的内存分配算法，它采用了分离适配的设计思想，将物理内存按照 2 的次幂进行划分，内存分配时也是按照 2 的次幂大小进行按需分配，例如 4KB、 8KB、16KB 等。假设我们请求分配的内存大小为 10KB，那么会按照 16KB 分配。

##### Slab 算法

#### jemalloc 架构设计



#### Netty 高性能内存设计

Netty 高性能的内存管理是借鉴 jemalloc 实现的，它同样需要解决两个经典的核心问题：

* 在单线程或者多线程的场景下，如何高效地进行内存分配和回收？
* 如何减少内存碎片，提高内存的有效利用率？

Netty 抽象出一些核心组件，如 PoolArena、PoolChunk、PoolChunkList、PoolSubpage、PoolThreadCache、MemoryRegionCache

