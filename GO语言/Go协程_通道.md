Go调度器：协程
------

### 线程调度

> Go 语言调度器：
>
> Go1.1：任务窃取处理器
>
> GO1.2 ：抢占式调度器:(基于协作的抢占，基于信号的抢占)
>
> Go 语言社区目前还有一个非均匀存储访问（Non-uniform memory access，NUMA）调度器的提案

N:1模型，多个用户空间线程在1个内核空间线程上运行。优势是上下文切换非常快，因为这些线程都在内核态运行，但是无法利用多核系统的优点。
		1:1模型，1个内核空间线程运行一个用户空间线程。这种充分利用了多核系统的优势但是上下文切换非常慢，因为每一次调度都会在用户态和内核态之间切换。		POSIX线程模型(pthread)就是这么做的。
		M:N模型，内核空间开启多个内核线程，一个内核空间线程对应多个用户空间线程。效率非常高，但是管理复杂。

### goroutine调度原理

图解协程：https://www.cnblogs.com/secondtonone1/p/11803961.html

golang采用了MPG模型管理协程，更加高效，但是管理非常复杂。
M：内核级线程
G：代表一个goroutine，一个待执行的任务；
P：Processor，处理器，用来管理和执行goroutine的；

#### G

Goroutine 在 Go 语言运行时使用私有结构体 `runtime.g` 表示，一共有四十多个变量：

> `stack` 字段描述了当前 Goroutine 的栈内存范围, `stackguard0` 可以用于调度器抢占式调度
>
> 每一个 Goroutine 上都持有两个分别存储 `defer` 和 `panic` 对应结构体的链表
>
> - `m` — 当前 Goroutine 占用的线程，可能为空；
>
> - `atomicstatus` — Goroutine 的状态；
>
> - `sched` — 存储 Goroutine 的调度相关的数据
>
> - `goid` — Goroutine 的 ID
>
>   ```go
>   //sched   gobuf 数据
>   type gobuf struct {
>   	sp   uintptr //栈指针
>   	pc   uintptr //程序计数器
>   	g    guintptr//持有的 Goroutine
>   	ret  sys.Uintreg //系统调用的返回值
>   	...
>   }
>   ```
>
>   **Goroutine 的状态**: 
>
>   - 等待中：Goroutine 正在等待某些条件满足，例如：系统调用结束等，包括 `_Gwaiting`、`_Gsyscall` 和 `_Gpreempted` 几个状态；
>   - 可运行：Goroutine 已经准备就绪，可以在线程运行，如果当前程序中有非常多的 Goroutine，每个 Goroutine 就可能会等待更多的时间，即 `_Grunnable`；
>   - 运行中：Goroutine 正在某个线程上运行，即 `_Grunning`；

#### M

Go 语言并发模型中的 M 是操作系统线程。调度器最多可以创建 10000 个线程，但是其中大多数的线程都不会执行用户代码（可能陷入系统调用），最多只会有 `GOMAXPROCS` 个活跃线程能够正常运行。在默认情况下，运行时会将 `GOMAXPROCS` 设置成当前机器的核数，

> Go 语言会使用私有结构体 [`runtime.m`](https://draveness.me/golang/tree/runtime.m) 表示操作系统线程
>
> 

### P

调度器中的处理器 P 是线程和 Goroutine 的中间层，它能提供线程需要的上下文环境，也会负责调度线程上的等待队列，通过处理器 P 的调度，每一个内核线程都能够执行多个 Goroutine，它能在 Goroutine 进行一些 I/O 操作时及时让出计算资源，提高线程的利用率。

[`runtime.p`](https://draveness.me/golang/tree/runtime.p) 是处理器的运行时表示，作为调度器的内部实现，它包含的字段也非常多，其中包括与**性能追踪**、**垃圾回收**和**计时器**相关的字段

> `runtime.p`结构体中的状态 `status` 字段会是以下五种中的一种：
>
> ​	

### Go调度器原理

##### 1.调度器初始化

1. 如果全局变量 `allp` 切片中的处理器数量少于期望数量，会对切片进行扩容；
2. 使用 `new` 创建新的处理器结构体并调用 [`runtime.p.init`](https://draveness.me/golang/tree/runtime.p.init) 初始化刚刚扩容的处理器；
3. 通过指针将线程 m0 和处理器 `allp[0]` 绑定到一起；
4. 调用 [`runtime.p.destroy`](https://draveness.me/golang/tree/runtime.p.destroy) 释放不再使用的处理器结构；
5. 通过截断改变全局变量 `allp` 的长度保证与期望处理器数量相等；
6. 将除 `allp[0]` 之外的处理器 P 全部设置成 `_Pidle` 并加入到全局的空闲队列中；

##### 2.创建goroutine

初始化结构体

将 Goroutine 放到运行队列上

设置调度信息

##### 3.调度循环

[`runtime.schedule`](https://draveness.me/golang/tree/runtime.schedule) 函数会从下面几个地方查找待执行的 Goroutine：

1. 为了保证公平，当全局运行队列中有待执行的 Goroutine 时，通过 `schedtick` 保证有一定几率会从全局的运行队列中查找对应的 Goroutine；
2. 从处理器本地的运行队列中查找待执行的 Goroutine；
3. 如果前两种方法都没有找到 Goroutine，会通过 [`runtime.findrunnable`](https://draveness.me/golang/tree/runtime.findrunnable) 进行阻塞地查找 Goroutine；

##### 4.触发调度

<img src="assets/2020-02-05-15808864354679-schedule-points.png" alt="schedule-points" style="zoom:67%;" />

- 主动挂起 — [`runtime.gopark`](https://draveness.me/golang/tree/runtime.gopark) -> [`runtime.park_m`](https://draveness.me/golang/tree/runtime.park_m)
- 系统调用 — [`runtime.exitsyscall`](https://draveness.me/golang/tree/runtime.exitsyscall) -> [`runtime.exitsyscall0`](https://draveness.me/golang/tree/runtime.exitsyscall0)
- 协作式调度 — [`runtime.Gosched`](https://draveness.me/golang/tree/runtime.Gosched) -> [`runtime.gosched_m`](https://draveness.me/golang/tree/runtime.gosched_m) -> [`runtime.goschedImpl`](https://draveness.me/golang/tree/runtime.goschedImpl)
- 系统监控 — [`runtime.sysmon`](https://draveness.me/golang/tree/runtime.sysmon) -> [`runtime.retake`](https://draveness.me/golang/tree/runtime.retake) -> [`runtime.preemptone`](https://draveness.me/golang/tree/runtime.preemptone)