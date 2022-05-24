### 基础

##### Go和java

> Java语言优势：
>
> 1.多线程编程  2.热修复补丁功能  3.内存管理机制 
>
> Go优势：1.协程2.CSP并发模型 3.

##### golang 中 make 和 new 的区别？（基本必问）

>   <u>new 只分配内存，而 make 只能用于 slice、map 和 channel 的初始化</u>
>
>   **new :** func new(Type) *Type 接收new 函数只接受一个参数，这个参数是一个类型，并且**返回一个指向该类型内存地址的指针**。同时 new 函数会把分配的内存置为零，也就是类型的零值。
>
>   make : make 函数只用于 map，slice 和 channel，并且不返回指针。如果想要获得一个显式的指针，可以使用 new 函数进行分配，或者显式地使用一个变量的地址。使用make创建map `map1 := make(map[int]float32) `

##### 数组和切片的区别 （基本必问）

>   数组固定大小，切片动态扩展其大小，多个切片底层共用一个数组，如果更改一个，其他的也都会更改，注意是[start,end-1]
>
>   创建切片的四种方式：

##### for range 的时候它的地址会发生变化么？

>   无论是经典的 for 循环还是 for-range 循环都会使用 `JMP` 等命令跳回循环体的开始位置复用代码。使用 for-range 的控制结构最终也会被 Go 语言编译器转换成普通的 for 循环
>
>   

4、go defer，多个 defer 的顺序，defer 在什么时机会修改返回值？

5、 uint 类型溢出

6、介绍 rune 类型

7、 golang 中解析 tag 是怎么实现的？反射原理是什么？

8、调用函数传入结构体时，应该传值还是指针？ （Golang 都是传值）

##### 协程和线程的区别

>   **调度：**OS的线程由OS内核调度，每隔⼏毫秒，⼀个硬件时钟中断发到CPU，CPU调⽤⼀个调度器内核函数。这个函数暂停当前正在运⾏的线程，把他的寄存器信息保存到内存中，查看线程列表并决定接下来运⾏哪⼀个线程，再从内存中恢复线程的注册表信息，最后继续执⾏选中的线程。这种线程切换需要⼀个完整的上下⽂切换；而GO协程实现了自己的一个调度器，调度器不是由硬件时钟来定期触发的，不需要上下文切换的开销。
>
>   栈空间上：OS线程都有⼀个固定⼤⼩的栈内存，通常是2MB，用来保存在其他函数调⽤期间哪些正在执⾏或者临时暂停的函数的局部变量；协程在在⽣命周期开始只有⼀个很⼩的栈，典型情况是2KB，而且可以按需增⼤和缩⼩，最⼤限制可以到1GB

### context相关

1、context 结构是什么样的？

> 接口，有四个方法，四个实现，分别有不同的功能，

2、context 使用场景和用途



### channel相关

1、channel 是否线程安全？锁用在什么地方？

2、go channel 的底层实现原理 （数据结构）

3、nil、关闭的 channel、有数据的 channel，再进行读、写、关闭会怎么样？（各类变种题型）

4、向 channel 发送数据和从 channel 读数据的流程是什么样的？

### map相关

1.map 使用注意的点，并发安全？

2.map 循环是有序的还是无序的？

3、 map 中删除一个 key，它的内存会释放么？

4、怎么处理对 map 进行并发访问？有没有其他方案？ 区别是什么？

5、 nil map 和空 map 有何不同？

6、map 的数据结构是什么？是怎么实现扩容？

###  GMP相关

1.   什么是 GMP？（必问）

     >   Golang内部有三个对象： P**对象(processor) 代表上下⽂**（或者可以认为是cpu），M(work thread)代表**⼯作线程**，G对象（**goroutine**）.
     >

2、进程、线程、协程有什么区别？

3、抢占式调度是如何抢占的？

4、M 和 P 的数量问题？

### 锁相关

##### Go有那些琐

>   Go中的三种锁包括:**互斥锁**,**读写锁**,**sync.Map的安全的锁**.
>
>   ###### 互斥锁
>
>   ```go
>   var mutex sync.Mutex
>   func Write() {
>    mutex.Lock()
>    defer mutex.Unlock()
>   }
>   ```
>
>   ##### 读写锁
>
>   ```go
>   // RWMutex是⼀个读/写互斥锁，可以由任意数量的读操作或单个写操作持有。
>   // RWMutex的零值是未锁定的互斥锁。
>   //⾸次使⽤后，不得复制RWMutex。
>   //如果goroutine持有RWMutex进⾏读取⽽另⼀个goroutine可能会调⽤Lock，那么在释放初始读锁之前，
>   goroutine不应该期望能够获取读锁定。
>   //特别是，这种禁⽌递归读锁定。 这是为了确保锁最终变得可⽤; 阻⽌的锁定会阻⽌新读操作获取锁定。
>   type RWMutex struct {
>    w Mutex //如果有待处理的写操作就持有
>    writerSem uint32 // 写操作等待读操作完成的信号量
>    readerSem uint32 //读操作等待写操作完成的信号量
>    readerCount int32 // 待处理的读操作数量
>    readerWait int32 // number of departing readers
>   }
>   ```
>
>   ##### sync.Map安全锁
>
>   ```go
>    type Map struct {
>    // 该锁⽤来保护dirty
>    mu Mutex
>    // 存读的数据，因为是atomic.value类型，只读类型，所以它的读是并发安全的
>    read atomic.Value // readOnly
>    //包含最新的写⼊的数据，并且在写的时候，会把read 中未被删除的数据拷⻉到该dirty中，因为是普通的map
>   存在并发安全问题，需要⽤到上⾯的mu字段。
>    dirty map[interface{}]*entry
>    // 从read读数据的时候，会将该字段+1，当等于len（dirty）的时候，会将dirty拷⻉到read中（从⽽提升
>   读的性能）。
>    misses int
>   }
>   ```
>
>   

1、除了 mutex 以外还有那些方式安全读写共享变量？

2、Go 如何实现原子操作？

3、Mutex 是悲观锁还是乐观锁？[悲观锁](https://www.zhihu.com/search?q=悲观锁&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"article"%2C"sourceId"%3A"434629143"})、乐观锁是什么？

4、Mutex 有几种模式？

5、goroutine 的自旋占用资源如何解决

###  并发相关

1.   Golang中常⽤的并发模型？

     >   1.   通过channel通知实现并发控制
     >
     >   2.   通过sync包中的WaitGroup实现并发控制（）
     >
     >        在 sync 包中，提供了 WaitGroup ，它会等待它收集的所有 goroutine 任务全部完成。在WaitGroup⾥主要有三个⽅法:Add, 可以添加或减少goroutine的数量.Done, 相当于Add(-1).Wait, 执⾏后会堵塞主线程，直到WaitGroup ⾥的值减⾄0.不能拷贝WaitGroup ，可能会造成死锁
     >        
     >   3.   在Go 1.7 以后引进的强⼤的Context上下⽂，实现并发控制
     >
     >        >   context 包主要是⽤来处理多个 goroutine 之间共享数据，及多个 goroutine 的管理。
     
     

1、怎么控制并发数？

2、多个 goroutine 对同一个 map 写会 panic，异常是否可以用 defer 捕获？

3、如何优雅的实现一个 goroutine 池（百度、手写代码）

### GC相关

1、go gc 是怎么实现的？（必问）

>   

2、go 是 gc 算法是怎么实现的？ （得物，出现频率低）

>   垃圾回收有三个阶段：_GCoff `、`_GCmark` 和 `_GCmarktermination

3、GC 中 stw 时机，各个阶段是如何解决的？ （百度）

>   

4、GC 的触发时机？

>   1.   后台触发
>
>   运行时会在应用程序启动时在后台开启一个用于强制触发垃圾收集的 Goroutine，该 Goroutine 的职责非常简单 — 调用 [`runtime.gcStart`](https://draveness.me/golang/tree/runtime.gcStart) 尝试启动新一轮的垃圾收集：
>
>   2.   手动触发
>
>        ```go
>        func GC() {
>        	n := atomic.Load(&work.cycles)
>        	gcWaitOnMark(n)//等待上一个循环的标记终止、标记和清除终止阶段完成；
>          //触发新一轮的垃圾收集并通过 runtime.gcWaitOnMark 等待该轮垃圾收集的标记终止阶段正常结束；
>        	gcStart(gcTrigger{kind: gcTriggerCycle, n: n + 1})
>        	gcWaitOnMark(n + 1)
>        // 清理全部待处理的内存管理单元并等待所有的清理工作完成，等待期间会调用 runtime.Gosched 让出处理器；
>        	for atomic.Load(&work.cycles) == n+1 && sweepone() != ^uintptr(0) {
>        		sweep.nbgsweep++
>        		Gosched()
>        	}
>                  
>        	for atomic.Load(&work.cycles) == n+1 && atomic.Load(&mheap_.sweepers) != 0 {
>        		Gosched()
>        	}
>                  
>        	mp := acquirem()
>        	cycle := atomic.Load(&work.cycles)
>        	if cycle == n+1 || (gcphase == _GCmark && cycle == n+2) {
>            //将该阶段的堆内存状态快照发布出来，我们可以获取这时的内存状态；
>        		mProf_PostSweep()
>        	}
>        	releasem(mp)
>        }
>        ```
>
>        3.   申请内存
>
>        

### 内存相关

1、谈谈内存泄露，什么情况下内存会泄露？怎么定位排查内存泄漏问题？

2、知道 golang 的内存逃逸吗？什么情况下会发生内存逃逸？

3、请简述 Go 是如何分配内存的？

>   Channel 分配在栈上还是堆上？哪些对象分配在堆上，哪些对象分配在栈上？

4、介绍一下大对象小对象，为什么小对象多了会造成 gc 压力？

### 协程相关

1.   Goroutine和线程的区别

     >   				1.   Go运⾏的时候包涵⼀个⾃⼰的调度器，这个调度器使⽤⼀个称为⼀个M:N调度技术，m个goroutine到n个os线程（可以⽤GOMAXPROCS来控制n的数量），Go的调度器不是由硬件时钟来定期触发的，⽽是由特定的go语⾔结构来触发的，**他不需要切换到内核语境**，所以调度⼀个goroutine⽐调度⼀个线程的成本低很多。
     >					
     >   		2. 每个OS的线程都有⼀个固定⼤⼩的栈内存，通常是2MB，栈内存⽤于保存在其他函数调⽤期间哪些正在执⾏或者临时暂停的函数的局部变量。这个固定的栈⼤⼩，如果对于goroutine来说，可能是⼀种巨⼤的浪费。作为对⽐goroutine在⽣命周期开始**只有⼀个很⼩的栈，典型情况是2KB**, 在go程序中，⼀次创建⼗万左右的goroutine也不罕⻅（2KB*100,000=200MB）。⽽且goroutine的栈不是固定⼤⼩，它可以按需增⼤和缩⼩，最⼤限制可以到1GB
     >   		2. 可以由语⾔和框架层进⾏调度。

2.   启动和停止协程


### 锁相关

Go中的三种锁包括:互斥锁(Mutex),读写锁(RWMutex),安全锁(sync.Map).



### 网络相关



### 其他

##### defer 的作⽤

1.   defer 语句经常被⽤于处理成对的操作，如打开、关闭、连接、断开连接、加锁、释放锁。

2.   通过 defer 机制，不论函数逻辑多复杂，都能保证在任何执⾏路径下，资源被释放。

3.   释放资源的 defer 应该直接跟在请求资源的语句后。

