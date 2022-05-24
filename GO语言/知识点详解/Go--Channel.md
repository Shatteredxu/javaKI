通道Channel
-----------

### CSP模型

通道的主要作用是用来**实现并发同步**，最初的设计原则就是：不要让计算通过共享内存来通讯(如果通过共享内存来通讯，则需要传统的并发同步技术（比如互斥锁）来避免数据竞争)，而应该让它们通过通讯来共享内存（随着一个数据值的传递（发送和接收），数据值的所有权从一个协程转移到了另一个协程.

大多数的编程语言的并发编程模型是<u>基于线程和内存同步访问控制</u>，Go 的并发编程的模型则用 <u>goroutine 和 channel</u> 来替代。Goroutine 和线程类似，channel 和 mutex (用于内存同步访问控制)类似。

通道的主要作用是用来控制并发，使两个协程通过通道来通信，当一 个协程发送一个值到一个通道，我们可以认为此协程释放了一些值的所有权。 当一个协程从一个通道接收到一个值，我们可以认为此协程获取了一些值的所有权。

### 通道操作

make创建: `ch := make(chan int)`,`make(chan int, 100)`

通道类型的零值也使用预声明的nil来表示。 一个非零通道值必须通过内置的make函数来创建。

关闭通道：close(ch) 

通道ch发送值v ：ch <- v 

从通道ch接收一个值 <–ch

查询通道的容量cap(ch)

查询通道的长度len(ch) 

| 操作     | nil channel  | closed channel     | not nil, not closed channel                                  |
| -------- | ------------ | ------------------ | ------------------------------------------------------------ |
| close    | <u>panic</u> | <u>panic</u>       | 正常关闭                                                     |
| 读 <- ch | 阻塞         | 读到对应类型的零值 | 阻塞或正常读取数据。缓冲型 channel 为空或非缓冲型 channel 没有等待发送者时会阻塞 |
| 写 ch <- | 阻塞         | <u>panic</u>       | 阻塞或正常写入数据。非缓冲型 channel 没有等待接收者或缓冲型 channel buf 满时会被阻塞 |



### 原理

##### 数据结构

```golang
type hchan struct {
	// chan 里元素数量
	qcount   uint
	// chan 底层循环数组的长度
	dataqsiz uint
	// 指向底层循环数组的指针
	// 只针对有缓冲的 channel
	buf      unsafe.Pointer
	// chan 中元素大小
	elemsize uint16
	// chan 是否被关闭的标志
	closed   uint32
	// chan 中元素类型
	elemtype *_type // element type
	// 已发送元素在循环数组中的索引
	sendx    uint   // send index
	// 已接收元素在循环数组中的索引
	recvx    uint   // receive index
	// 等待接收的 goroutine 队列
	recvq    waitq  // list of recv waiters
	// 等待发送的 goroutine 队列
	sendq    waitq  // list of send waiters
	// 保护 hchan 中所有字段
	lock mutex
}
type waitq struct {
	first *sudog
	last  *sudog
}
```

通过makeChan返回一个指针, 分为有缓存区和无缓冲区的内存分配

```go
const hchanSize = unsafe.Sizeof(hchan{}) + uintptr(-int(unsafe.Sizeof(hchan{}))&(maxAlign-1))

func makechan(t *chantype, size int64) *hchan {
	elem := t.elem

	// 省略了检查 channel size，align 的代码
	// ……

	var c *hchan
	// 如果元素类型不含指针 或者 size 大小为 0（无缓冲类型）
	// 只进行一次内存分配
	if elem.kind&kindNoPointers != 0 || size == 0 {
		// 如果 hchan 结构体中不含指针，GC 就不会扫描 chan 中的元素
		// 只分配 "hchan 结构体大小 + 元素大小*个数" 的内存
		c = (*hchan)(mallocgc(hchanSize+uintptr(size)*elem.size, nil, true))
		// 如果是缓冲型 channel 且元素大小不等于 0（大小等于 0的元素类型：struct{}）
		if size > 0 && elem.size != 0 {
			c.buf = add(unsafe.Pointer(c), hchanSize)
		} else {
			// race detector uses this location for synchronization
			// Also prevents us from pointing beyond the allocation (see issue 9401).
			// 1. 非缓冲型的，buf 没用，直接指向 chan 起始地址处
			// 2. 缓冲型的，能进入到这里，说明元素无指针且元素类型为 struct{}，也无影响
			// 因为只会用到接收和发送游标，不会真正拷贝东西到 c.buf 处（这会覆盖 chan的内容）
			c.buf = unsafe.Pointer(c)
		}
	} else {
		// 进行两次内存分配操作
		c = new(hchan)
		c.buf = newarray(elem, int(size))
	}
	c.elemsize = uint16(elem.size)
	c.elemtype = elem
	// 循环数组长度
	c.dataqsiz = uint(size)
	// 返回 hchan 指针
	return c
}
```

##### 数据发送操作

1.   首先查看channel是否为空
2.   在查看有没有多余的空间来接受数据
3.   上锁，判断接受队列有没有线程等待，有就直接把数据交给协程
4.   没有就判断缓存区有没有位置，没有报错，有则将数据复制进去
5.   channel 满了，发送方会被阻塞。接下来会构造一个 sudog，放入发送阻塞队列，阻塞协程
6.   等待协程被唤醒，

```golang
// 位于 src/runtime/chan.go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	// 如果 channel 是 nil
	if c == nil {
		// 不能阻塞，直接返回 false，表示未发送成功
		if !block {
			return false
		}
		// 当前 goroutine 被挂起
		gopark(nil, nil, "chan send (nil chan)", traceEvGoStop, 2)
		throw("unreachable")
	}

	// 省略 debug 相关……

	// 对于不阻塞的 send，快速检测失败场景
	//
	// 如果 channel 未关闭且 channel 没有多余的缓冲空间。这可能是：
	// 1. channel 是非缓冲型的，且等待接收队列里没有 goroutine
	// 2. channel 是缓冲型的，但循环数组已经装满了元素
	if !block && c.closed == 0 && ((c.dataqsiz == 0 && c.recvq.first == nil) ||
		(c.dataqsiz > 0 && c.qcount == c.dataqsiz)) {
		return false
	}
	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}
	// 锁住 channel，并发安全
	lock(&c.lock)

	// 如果 channel 关闭了
	if c.closed != 0 {
		// 解锁
		unlock(&c.lock)
		// 直接 panic
		panic(plainError("send on closed channel"))
	}

	// 如果接收队列里有 goroutine，直接将要发送的数据拷贝到接收 goroutine
	if sg := c.recvq.dequeue(); sg != nil {
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}

	// 对于缓冲型的 channel，如果还有缓冲空间
	if c.qcount < c.dataqsiz {
		// qp 指向 buf 的 sendx 位置
		qp := chanbuf(c, c.sendx)

		// ……
		// 将数据从 ep 处拷贝到 qp
		typedmemmove(c.elemtype, qp, ep)
		// 发送游标值加 1
		c.sendx++
		// 如果发送游标值等于容量值，游标值归 0
		if c.sendx == c.dataqsiz {
			c.sendx = 0
		}
		// 缓冲区的元素数量加一
		c.qcount++

		// 解锁
		unlock(&c.lock)
		return true
	}

	// 如果不需要阻塞，则直接返回错误
	if !block {
		unlock(&c.lock)
		return false
	}

	// channel 满了，发送方会被阻塞。接下来会构造一个 sudog

	// 获取当前 goroutine 的指针
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}

	mysg.elem = ep
	mysg.waitlink = nil
	mysg.g = gp
	mysg.selectdone = nil
	mysg.c = c
	gp.waiting = mysg
	gp.param = nil
	// 当前 goroutine 进入发送等待队列
	c.sendq.enqueue(mysg)
	// 当前 goroutine 被挂起
	goparkunlock(&c.lock, "chan send", traceEvGoBlockSend, 3)

	// 从这里开始被唤醒了（channel 有机会可以发送了）
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	if gp.param == nil {
		if c.closed == 0 {
			throw("chansend: spurious wakeup")
		}
		// 被唤醒后，channel 关闭了。坑爹啊，panic
		panic(plainError("send on closed channel"))
	}
	gp.param = nil
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	// 去掉 mysg 上绑定的 channel
	mysg.c = nil
	releaseSudog(mysg)
	return true
}
```

##### 数据接受操作

接受类型的chan返回值分为两种：1. 返回ok和不返回ok，但跟进源码都是调用chanrecv

```golang
// 位于 src/runtime/chan.go
// chanrecv 函数接收 channel c 的元素并将其写入 ep 所指向的内存地址。
// 如果 ep 是 nil，说明忽略了接收值。
// 如果 block == false，即非阻塞型接收，在没有数据可接收的情况下，返回 (false, false)
// 否则，如果 c 处于关闭状态，将 ep 指向的地址清零，返回 (true, false)
// 否则，用返回值填充 ep 指向的内存地址。返回 (true, true)
// 如果 ep 非空，则应该指向堆或者函数调用者的栈

func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	// 省略 debug 内容 …………

	// 如果是一个 nil 的 channel
	if c == nil {
		// 如果不阻塞，直接返回 (false, false)
		if !block {
			return
		}
		// 否则，接收一个 nil 的 channel，goroutine 挂起
		gopark(nil, nil, "chan receive (nil chan)", traceEvGoStop, 2)
		// 不会执行到这里
		throw("unreachable")
	}

	// 在非阻塞模式下，快速检测到失败，不用获取锁，快速返回
	// 当我们观察到 channel 没准备好接收：
	// 1. 非缓冲型，等待发送列队 sendq 里没有 goroutine 在等待
	// 2. 缓冲型，但 buf 里没有元素
	// 之后，又观察到 closed == 0，即 channel 未关闭。
	// 因为 channel 不可能被重复打开，所以前一个观测的时候 channel 也是未关闭的，
	// 因此在这种情况下可以直接宣布接收失败，返回 (false, false)
	if !block && (c.dataqsiz == 0 && c.sendq.first == nil ||
		c.dataqsiz > 0 && atomic.Loaduint(&c.qcount) == 0) &&
		atomic.Load(&c.closed) == 0 {
		return
	}

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}

	// 加锁
	lock(&c.lock)

	// channel 已关闭，并且循环数组 buf 里没有元素
	// 这里可以处理非缓冲型关闭 和 缓冲型关闭但 buf 无元素的情况
	// 也就是说即使是关闭状态，但在缓冲型的 channel，
	// buf 里有元素的情况下还能接收到元素
	if c.closed != 0 && c.qcount == 0 {
		if raceenabled {
			raceacquire(unsafe.Pointer(c))
		}
		// 解锁
		unlock(&c.lock)
		if ep != nil {
			// 从一个已关闭的 channel 执行接收操作，且未忽略返回值
			// 那么接收的值将是一个该类型的零值
			// typedmemclr 根据类型清理相应地址的内存
			typedmemclr(c.elemtype, ep)
		}
		// 从一个已关闭的 channel 接收，selected 会返回true
		return true, false
	}

	// 等待发送队列里有 goroutine 存在，说明 buf 是满的
	// 这有可能是：
	// 1. 非缓冲型的 channel
	// 2. 缓冲型的 channel，但 buf 满了
	// 针对 1，直接进行内存拷贝（从 sender goroutine -> receiver goroutine）
	// 针对 2，接收到循环数组头部的元素，并将发送者的元素放到循环数组尾部
	if sg := c.sendq.dequeue(); sg != nil {
		// Found a waiting sender. If buffer is size 0, receive value
		// directly from sender. Otherwise, receive from head of queue
		// and add sender's value to the tail of the queue (both map to
		// the same buffer slot because the queue is full).
		recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true, true
	}

	// 缓冲型，buf 里有元素，可以正常接收
	if c.qcount > 0 {
		// 直接从循环数组里找到要接收的元素
		qp := chanbuf(c, c.recvx)

		// …………

		// 代码里，没有忽略要接收的值，不是 "<- ch"，而是 "val <- ch"，ep 指向 val
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		// 清理掉循环数组里相应位置的值
		typedmemclr(c.elemtype, qp)
		// 接收游标向前移动
		c.recvx++
		// 接收游标归零
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		// buf 数组里的元素个数减 1
		c.qcount--
		// 解锁
		unlock(&c.lock)
		return true, true
	}

	if !block {
		// 非阻塞接收，解锁。selected 返回 false，因为没有接收到值
		unlock(&c.lock)
		return false, false
	}

	// 接下来就是要被阻塞的情况了
	// 构造一个 sudog
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}

	// 待接收数据的地址保存下来
	mysg.elem = ep
	mysg.waitlink = nil
	gp.waiting = mysg
	mysg.g = gp
	mysg.selectdone = nil
	mysg.c = c
	gp.param = nil
	// 进入channel 的等待接收队列
	c.recvq.enqueue(mysg)
	// 将当前 goroutine 挂起
	goparkunlock(&c.lock, "chan receive", traceEvGoBlockRecv, 3)

	// 被唤醒了，接着从这里继续执行一些扫尾工作
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	closed := gp.param == nil
	gp.param = nil
	mysg.c = nil
	releaseSudog(mysg)
	return true, !closed
}
```

##### 关闭操作



### 通道使用场景

Channel 和 goroutine 的结合是 Go 并发编程的大杀器。而 Channel 的实际应用也经常让人眼前一亮，通过与 select，cancel，timer 等结合，它能实现各种各样的功能。接下来，我们就要梳理一下 channel 的应用。

1.   停止信号

2.   任务定时（与timer结合）

     >   超时控制：
     >
     >   ```go
     >   select {
     >   	case <-time.After(100 * time.Millisecond):
     >   	case <-s.stopc:
     >   		return false
     >   }
     >   ```
     >
     >   定时任务：
     >
     >   ```golang
     >   func worker() {
     >   	ticker := time.Tick(1 * time.Second)
     >   	for {
     >   		select {
     >   		case <- ticker:
     >   			// 执行定时任务
     >   			fmt.Println("执行 1s 定时任务")
     >   		}
     >   	}
     >   }
     >   ```

3.   解耦生产方和消费方

     ```go
     // 生产者: 生成 factor 整数倍的序列
     func Producer(factor int, out chan<- int) {
         for i := 0; ; i++ {
             out <- i*factor
         }
     }
     
     // 消费者
     func Consumer(in <-chan int) {
         for v := range in {
             fmt.Println(v)
         }
     }
     func main() {
         ch := make(chan int, 64) // 成果队列
     
         go Producer(3, ch) // 生成 3 的倍数的序列
         go Producer(5, ch) // 生成 5 的倍数的序列
         go Consumer(ch)    // 消费 生成的队列
     
         // 运行一定时间后退出
         time.Sleep(5 * time.Second)
     }
     ```

4.   控制并发数

     ```golang
     var limit = make(chan int, 3)
     
     func main() {
         // …………
         for _, w := range work {
             go func() {
                 limit <- 1//加锁
                 w()
                 <-limit
             }()
         }
         // …………
     }
     ```

     

##### 用做future/promise

##### 使用通道实现通知

### 面试题

1.   从一个被关闭的Channel中能读到数据吗

     >   答案是可以，如果通道里面还有值，则可以读到，返回值ok为true
     >
     >   ```golang
     >   func main() {
     >   	ch := make(chan int, 5)
     >   	ch <- 18
     >   	close(ch)
     >   	x, ok := <-ch
     >   	if ok {
     >   		fmt.Println("received: ", x)
     >   	}
     >   
     >   	x, ok = <-ch
     >   	if !ok {
     >   		fmt.Println("channel closed, data invalid.")
     >   	}
     >   }
     >   ```

2.   如何优雅的关闭Channel

     >   Channel 使用过程中不方便的地方：
     >
     >   1.  在不改变 channel 自身状态的情况下，无法获知一个 channel 是否关闭。
     >   2.  关闭一个 closed channel 会导致 panic。所以，如果关闭 channel 的一方在不知道 channel 是否处于关闭状态时就去贸然关闭 channel 是很危险的事情。
     >   3.  向一个 closed channel 发送数据会导致 panic。所以，如果向 channel 发送数据的一方不知道 channel 是否处于关闭状态时就去贸然向 channel 发送数据是很危险的事情。
     >
     >   所以如何不危险的关闭Channel：
     >
     >   1.  一个 sender，一个 receiver
     >
     >   2.  一个 sender， M 个 receiver
     >
     >       //上面两种情况，直接在sender端进行关闭
     >
     >   3.  N 个 sender，一个 reciver
     >
     >   4.  N 个 sender， M 个 receiver
     >
     >       //使用一下方法
     >
     >   通过设置一个关闭的channel，当我们需要关闭的时候，则向关闭channel发送数据，收到数据则调用函数关闭，而关闭的channel则通过GC回收
     >
     >   ```
     >   func main() {
     >   	rand.Seed(time.Now().UnixNano())
     >   	const Max = 100000
     >   	const NumSenders = 1000
     >   	dataCh := make(chan int, 100)
     >   	stopCh := make(chan struct{})
     >   	// senders
     >   	for i := 0; i < NumSenders; i++ {
     >   		go func() {
     >   			for {
     >   				select {
     >   				case <- stopCh:
     >   					return
     >   				case dataCh <- rand.Intn(Max):
     >   				}
     >   			}
     >   		}()
     >   	}
     >   	// the receiver
     >   	go func() {
     >   		for value := range dataCh {
     >   			if value == Max-1 {
     >   				fmt.Println("send stop signal to senders.")
     >   				close(stopCh)
     >   				return
     >   			}
     >   
     >   			fmt.Println(value)
     >   		}
     >   	}()
     >   	select {
     >   	case <- time.After(time.Hour):
     >   	}
     >   }
     >   ```

3.   Channel 泄露

     goroutine 操作 channel 后，处于发送或接收阻塞状态，而 channel 处于满或空的状态，一直得不到改变。同时，垃圾回收器也不会回收此类资源，进而导致 gouroutine 会一直处于等待队列中，不见天日。

     另外，程序运行过程中，对于一个 channel，如果没有任何 goroutine 引用了，gc 会对其进行回收操作，不会引起内存泄漏。

     
