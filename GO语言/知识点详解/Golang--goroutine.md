Go调度器：协程
------

参考：https://learnku.com/articles/41728

任务调度模型：

N:1模型，多个用户空间线程在1个内核空间线程上运行。优势是上下文切换非常快，因为这些线程都在内核态运行，但是无法利用多核系统的优点。
		1:1模型，1个内核空间线程运行一个用户空间线程。这种充分利用了多核系统的优势但是上下文切换非常慢，因为每一次调度都会在用户态和内核态之间切换。		POSIX线程模型(pthread)就是这么做的。
		M:N模型，内核空间开启多个内核线程，一个内核空间线程对应多个用户空间线程。效率非常高，但是管理复杂。

### Go 语言调度器

0.x版本：单线程调度器。

1.0版本：多线程调度器

Go1.1：任务窃取处理器

>   在多线程的基础上：1. 添加了处理器 P层。 2. 在处理器 P 的基础上实现基于工作窃取的调度器。
>
>   

GO1.2 ：抢占式调度器:(基于协作的抢占，**基于信号的抢占**)

Go 语言社区目前还有一个非均匀存储访问（Non-uniform memory access，NUMA）调度器的提案、

### goroutine调度原理

图解协程：https://www.cnblogs.com/secondtonone1/p/11803961.html

golang采用了MPG模型管理协程，更加高效，但是管理非常复杂。
M：内核级线程
G：代表一个goroutine，一个待执行的任务；
P：Processor，处理器，调度器，用来管理和执行goroutine的；

#### G

Goroutine 在 Go 语言运行时使用私有结构体 `runtime.g` 表示，一共有四十多个变量：

> 1.   与栈相关的：`stack `当前线程的内存范围,`stackguard0 `用户抢占式调度。
> 2.   与抢占相关的：preempt       bool // 抢占信号，preemptStop   bool // 抢占时将状态修改成 `_Gpreempted`，preemptShrink bool // 在同步安全点收缩栈
>
> 3.   每一个 Goroutine 上都持有两个分别存储 `defer` 和 `panic` 对应结构体的链表
>
>      ```go
>      _panic       *_panic // 最内侧的 panic 结构体
>      _defer       *_defer // 最内侧的延迟函数结构体
>      ```
>
> 4.   
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
>   //这些内容在调度器保存或者恢复上下文的时候用到，其中的栈指针和程序计数器会用来存储或者恢复寄存器中的值，改变程序即将执行的代码
>   ```
>
>   **atomicstatus表示了Goroutine 的状态**: 
>
>   - 等待中：Goroutine 正在等待某些条件满足，例如：系统调用结束等，包括 `_Gwaiting`、`_Gsyscall` 和 `_Gpreempted` 几个状态；
>   - 可运行：Goroutine 已经准备就绪，可以在线程运行，如果当前程序中有非常多的 Goroutine，每个 Goroutine 就可能会等待更多的时间，即 `_Grunnable`；
>   - 运行中：Goroutine 正在某个线程上运行，即 `_Grunning`；

#### M

<u>M的个数：在go程序启动的时候会设置为最大数量10000，SetMaxThreads 可以手动设置，M阻塞会创建新的M</u>

Go 语言并发模型中的 M 是操作系统线程。调度器最多可以创建 10000 个线程，但是其中大多数的线程都不会执行用户代码（可能陷入系统调用），最多只会有 `GOMAXPROCS` 个活跃线程能够正常运行。在默认情况下，运行时会将 `GOMAXPROCS` 设置成当前机器的核数，

> Go 语言会使用私有结构体 [`runtime.m`](https://draveness.me/golang/tree/runtime.m) 表示操作系统线程
>
> ```go
> type m struct {
> 	g0   *g
> 	curg *g
> 	...
> }
> ```
>
> g0 是持有调度栈的 Goroutine，`curg` 是在当前线程上运行的用户 Goroutine，g0是一个运行时中比较特殊的 Goroutine，它会**深度参与运行时的调度过程**，包括 Goroutine 的创建、大内存分配和 CGO 函数的执行。
>
> 

### P

P的个数：启动时环境变量 `$GOMAXPROCS` 或者是由 `runtime` 的方法 `GOMAXPROCS()` 决定

调度器中的处理器 P 是线程和 Goroutine 的中间层，它能提供线程需要的上下文环境，也会负责调度线程上的等待队列，通过处理器 P 的调度，每一个内核线程都能够执行多个 Goroutine，它能在 Goroutine 进行一些 I/O 操作时及时让出计算资源，提高线程的利用率。



[`runtime.p`](https://draveness.me/golang/tree/runtime.p) 是处理器的运行时表示，作为调度器的内部实现，它包含的字段也非常多，其中包括与**性能追踪**、**垃圾回收**和**计时器**相关的字段

```go
type p struct {
	m           muintptr

	runqhead uint32//持有的运行队列
	runqtail uint32//
	runq     [256]guintptr
	runnext guintptr//线程下一个需要执行的 Goroutine。
	...
}
```

> `runtime.p`结构体中的状态 `status` 字段会是以下五种中的一种：
>
> ​	

### Go调度器原理

##### 1.调度器初始化

在`schedinit`进行调度器初始化，完成相应数量处理器的启动，GOMAXPROCS规定了同时运行的最大处理器数。

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

刘丹冰视频
----------

1.   复用线程：

     >   work stealing 
     >
     >   hand off 机制：

2.   并行

3.   抢占，每个最多10ms，

4.   全局G队列

##### go function经历了什么

<img src="assets/a4vWtvRWGQ.jpeg!large" alt="18-go-func调度周期.jpeg" style="zoom:80%;" />

##### 生命周期

<img src="assets/j37FX8nek9.png!large" alt="17-pic-go调度器生命周期.png" style="zoom:70%;" />



**M0**：

M0 是启动程序后的编号为 0 的主线程，这个 M 对应的实例会在全局变量 runtime.m0 中，不需要在 heap 上分配，M0 负责执行初始化操作和启动第一个 G， 在之后 M0 就和其他的 M 一样了。

**G0**：

G0 是每次启动一个 M 都会第一个创建的 goroutine，G0 仅用于负责调度的 G，G0 不指向任何可执行的函数，每个 M 都会有一个自己的 G0。在调度或系统调用时会使用 G0 的栈空间，全局变量的 G0 是 M0 的 G0。

