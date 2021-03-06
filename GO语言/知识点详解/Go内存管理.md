编程语言的内存分配器一般包含两种分配方法，一种是**线性分配器**（Sequential Allocator，Bump Allocator），另一种是**空闲链表分配器**（Free-List Allocator）

### 线性分配器

线性分配器在内存中维护一个指向内存特定位置的指针，如果用户程序向分配器申请内存，分配器只需要检查剩余的空闲内存、返回分配的内存区域并修改指针在内存中的位置，它带来了较快的内存分配速度及较低的实现复杂度，但是线性分配器无法在内存被释放时重用内存(也就是已用内存和未用内存混杂在一起，不能分开)，所以需要与合适的垃圾回收算法配合使用，例如：<u>标记压缩（Mark-Compact）</u>、<u>复制回收（Copying GC）</u>和<u>分代回收（Generational GC）</u>等算法，它们可以通过拷贝的方式<u>整理存活对象的碎片</u>，将空闲内存定期合并，这也就是JVM的做法。

### 空闲链表分配器

<u>空闲链表分配器</u>在内部会维护一个类似**链表**的数据结构。当用户程序申请内存时，空闲链表分配器会依次遍历空闲的内存块，找到足够大的内存，然后申请新的资源并修改链表

**动态分配：**首次适应，循环首次适应，最优适应，隔离适应（将内存分割成多个链表，每个链表中的内存块大小相同，申请内存时先找到满足条件的链表，再从链表中选择合适的内存块）

伙伴算法

### 内存布局方式

线性内存：分配的堆内存是连续的，需要预留大块的内存空间

稀疏内存：稀疏内存是 Go 语言在 1.11 中提出的方案，使用稀疏的内存布局不仅能移除堆大小的上限，还能解决 C 和 Go 混合使用时的地址空间冲突问题。不过因为基于稀疏内存的内存管理失去了内存的连续性这一假设，这也使内存管理变得更加复杂

地址空间转换：也就是虚拟内存转换为物理地址，是由操作系统来完成的，运行时使用 Linux 提供的 `mmap`、`munmap` 和 `madvise` 等系统调用实现了操作系统的内存管理抽象层，抹平了不同操作系统的差异，为运行时提供了更加方便的接口

### 内存管理组件

Go 语言的内存分配器包含内存管理单元、线程缓存、中心缓存和页堆几个重要组件，分别对应[`runtime.mspan`](https://draveness.me/golang/tree/runtime.mspan)、[`runtime.mcache`](https://draveness.me/golang/tree/runtime.mcache)、[`runtime.mcentral`](https://draveness.me/golang/tree/runtime.mcentral) 和 [`runtime.mheap`](https://draveness.me/golang/tree/runtime.mheap)，

<img src="assets/2020-02-29-15829868066479-go-memory-layout.png" alt="go-memory-layout" style="zoom:67%;" />

#### 内存管理方式

`runtime.mspan `是 Go 语言内存管理的基本单元，该结构体中包含 next 和 prev 两个字段，它们分别指向了前一个和后一个 runtime.mspan：每个mspan都管理大小为8KB的若干个page页

`runtime.spanClass` 是 `runtime.mspan`的跨度类，它决定了内存管理单元中存储的对象大小和个数：对象大小从 8B 到 32KB，总共 67 种跨度类的大小、存储的对象数以及浪费的内存空间

##### 线程缓存

`runtime.mcache` 是 Go 语言中的线程缓存，它会与线程上的处理器一一绑定，主要用来缓存用户程序申请的微小对象。每一个线程缓存都持有 68 * 2 个 `runtime.mspan`，这些内存管理单元都存储在结构体的 `alloc` 字段中

##### 中心缓存

访问中心缓存中的内存管理单元需要使用互斥锁，对于每个线程都可以从这里进行内存申请，线程缓存会通过中心缓存的 `runtime.mcentral.cacheSpan`方法获取新的内存管理单元。

##### 页堆

`mheap`是内存分配的核心结构体，Go 语言程序会将其作为<u>全局变量存储</u>，而堆上初始化的所有对象都由该结构体统一管理，该结构体中包含两组非常重要的字段，其中一个是全局的中心缓存列表 `central`，另一个是管理堆区内存区域的 `arenas` 以及相关字段

#### 内存分配

堆上所有的对象都会通过调用 [`runtime.newobject`](https://draveness.me/golang/tree/runtime.newobject) 函数分配内存，该函数会调用 [`runtime.mallocgc`](https://draveness.me/golang/tree/runtime.mallocgc) 分配指定大小的内存空间，根据申请对象大小将它们分成<u>微对象、小对象和大对象</u>

*   微对象 `(0, 16B)` — 先使用微型分配器，再依次尝试线程缓存、中心缓存和堆分配内存；
*   小对象 `[16B, 32KB]` — 依次尝试使用线程缓存、中心缓存和堆分配内存；
*   大对象 `(32KB, +∞)` — 直接在堆上分配内存；

##### 微对象

小于 16 字节的对象划分为微对象，使用**线程缓存**上的微分配器提高微对象分配的性能，我们主要使用它来分配较小的字符串以及逃逸的临时变量

##### 小对象

小对象是指大小为 16 字节到 32,768 字节的对象以及所有小于 16 字节的指针类型的对象，小对象的分配可以被分成以下的三个步骤：

1.  确定分配对象的大小以及跨度类 `runtime.spanClass`；
2.  从线程缓存、中心缓存或者堆中获取内存管理单元并从内存管理单元找到空闲的内存空间；
3.  调用 runtime.memclrNoHeapPointers`清空空闲内存中的所有数据；

##### 大对象

运行时对于大于 32KB 的大对象会单独处理，我们不会从线程缓存或者中心缓存中获取内存管理单元，而是直接调用 `runtime.mcache.allocLarge` 分配大片内存：