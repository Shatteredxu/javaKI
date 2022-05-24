常見的同步原语 sync.Mutex、sync.RWMutex、sync.WaitGroup、sync.Once 和 sync.Cond 以及扩展原语 golang/sync/errgroup.Group、golang/sync/semaphore.Weighted 和 golang/sync/singleflight.Group

```

```

### sync.Mutex互斥锁

```go
type Mutex struct {
	state int32
	sema  uint32
}

```

对于state int32，由`mutexLocked`、`mutexWoken` 和 `mutexStarving`，`waiterCount`组成。

*   `mutexLocked` — 表示互斥锁的锁定状态；
*   `mutexWoken` — 表示从正常模式被从唤醒；
*   `mutexStarving` — 当前的互斥锁进入饥饿状态；
*   `waitersCount` — 当前互斥锁上等待的 Goroutine 个数；

##### 加锁

**正常模式和饥饿模式**，

1.   首先通过CompareAndSwapInt32去查看锁有没有被占用，没有则更改状态，获取到锁；，如果没有获取到则调用`lockSlow`方法

2.   lockSlow方法是一个大循环，

     >   1.  判断当前 Goroutine 能否进入自旋；
     >   2.  通过自旋等待互斥锁的释放；
     >   3.  计算互斥锁的最新状态；
     >   4.  更新互斥锁的状态并获取锁；
     >
     >   ```go
     >   func (m *Mutex) lockSlow() {
     >   	var waitStartTime int64
     >   	starving := false
     >   	awoke := false
     >   	iter := 0
     >   	old := m.state
     >   	for {
     >   	if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {//判断当前 Goroutine 能否进入自旋
     >   			if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
     >   				atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {//通过自旋等待互斥锁的释放；
     >   				awoke = true
     >   			}
     >   			runtime_doSpin()//执行 30 次的 PAUSE 指令，该指令只会占用 CPU 并消耗 CPU 时间
     >   			iter++
     >   			old = m.state
     >   			continue
     >   		}
     >   ```
     >
     >   互斥锁会根据上下文计算当前互斥锁最新的状态,计算state中的不同字段的值
     >
     >   ```go
     >   new := old
     >   		if old&mutexStarving == 0 {
     >   			new |= mutexLocked
     >   		}
     >   		if old&(mutexLocked|mutexStarving) != 0 {
     >   			new += 1 << mutexWaiterShift
     >   		}
     >   		if starving && old&mutexLocked != 0 {
     >   			new |= mutexStarving
     >   		}
     >   		if awoke {
     >   			new &^= mutexWoken
     >   		}
     >   //CAS更改为新的状态
     >   if atomic.CompareAndSwapInt32(&m.state, old, new) {
     >   			if old&(mutexLocked|mutexStarving) == 0 {
     >   				break // 通过 CAS 函数获取了锁
     >   			}
     >   			...
     >     //如果没有通过 CAS 获得锁，会调用 runtime.sync_runtime_SemacquireMutex 通过信号量保证资源不会被两个 Goroutine 获取。runtime.sync_runtime_SemacquireMutex 会在方法中不断尝试获取锁并陷入休眠等待信号量的释放，一旦当前 Goroutine 可以获取信号量，它就会立刻返回，sync.Mutex.Lock 的剩余代码也会继续执行
     >   			runtime_SemacquireMutex(&m.sema, queueLifo, 1)
     >   			starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
     >   			old = m.state
     >   			if old&mutexStarving != 0 {
     >   				delta := int32(mutexLocked - 1<<mutexWaiterShift)
     >   				if !starving || old>>mutexWaiterShift == 1 {
     >   					delta -= mutexStarving
     >   				}
     >   				atomic.AddInt32(&m.state, delta)
     >   				break
     >   			}
     >   			awoke = true
     >   			iter = 0
     >   		} else {
     >   			old = m.state
     >   		}
     >   	}
     >   }
     >   ```

##### 解锁

```go
func (m *Mutex) Unlock() {
	new := atomic.AddInt32(&m.state, -mutexLocked)//设置新的状态为0
	if new != 0 {
		m.unlockSlow(new)
	}
}
//没有成功设置为0，则unlockSlow，分为正常模式和饥饿模式
func (m *Mutex) unlockSlow(new int32) {
	if (new+mutexLocked)&mutexLocked == 0 {
		throw("sync: unlock of unlocked mutex")
	}
  //如果互斥锁不存在等待者或者互斥锁的 mutexLocked、mutexStarving、mutexWoken 状态不都为 0，那么当前方法可以直接返回，不需要唤醒其他等待者；
//如果互斥锁存在等待者，会通过 sync.runtime_Semrelease 唤醒等待者并移交锁的所有权；
	if new&mutexStarving == 0 { // 正常模式
		old := new
		for {
			if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
				return
			}
			new = (old - 1<<mutexWaiterShift) | mutexWoken
			if atomic.CompareAndSwapInt32(&m.state, old, new) {
				runtime_Semrelease(&m.sema, false, 1)
				return
			}
			old = m.state
		}
	} else { // 饥饿模式
    //在饥饿模式下，上述代码会直接调用 sync.runtime_Semrelease 将当前锁交给下一个正在尝试获取锁的等待者，等待者被唤醒后会得到锁，在这时互斥锁还不会退出饥饿状态
		runtime_Semrelease(&m.sema, true, 1)
	}
}
```

### 读写锁

### WaitGroup

```go
type WaitGroup struct {
	noCopy noCopy// 保证 sync.WaitGroup 不会被开发者通过再赋值的方式拷贝
	state1 [3]uint32//存储着状态和信号量
}


```

##### 接口

sync.WaitGroup 对外暴露了三个方法 — sync.WaitGroup.Add、sync.WaitGroup.Wait 和 sync.WaitGroup.Done。

```go
func (wg *WaitGroup) Add(delta int) {
	statep, semap := wg.state()
	state := atomic.AddUint64(statep, uint64(delta)<<32)
	v := int32(state >> 32)
	w := uint32(state)
	if v < 0 {
		panic("sync: negative WaitGroup counter")
	}
	if v > 0 || w == 0 {
		return
	}
	*statep = 0
	for ; w != 0; w-- {
		runtime_Semrelease(semap, false, 0)
	}
}
//会在计数器大于 0 并且不存在等待的 Goroutine 时，调用 runtime.sync_runtime_Semacquire 陷入睡眠
func (wg *WaitGroup) Wait() {
	statep, semap := wg.state()
	for {
		state := atomic.LoadUint64(statep)
		v := int32(state >> 32)
		if v == 0 {//当 sync.WaitGroup 的计数器归零时，陷入睡眠状态的 Goroutine 会被唤醒，上述方法也会立刻返回
			return
		}
		if atomic.CompareAndSwapUint64(statep, state, state+1) {
			runtime_Semacquire(semap)
			if +statep != 0 {
				panic("sync: WaitGroup is reused before previous Wait has returned")
			}
			return
		}
	}
}
```

### Cond

它可以让一组的 Goroutine 都在满足特定条件时被唤醒。每一个 `sync.Cond`结构体在初始化时都需要传入一个互斥锁.

```go
type Cond struct {
	noCopy  noCopy//用于保证结构体不会在编译期间拷贝
	L       Locker//用于禁止运行期间发生的拷贝
	notify  notifyList// 用于保护内部的 notify 字段，Locker 接口类型的变量
	checker copyChecker// 一个 Goroutine 的链表，它是实现同步机制的核心结构
}
type notifyList struct {
	wait uint32//wait 和 notify 分别表示当前正在等待的和已经通知到的 Goroutine 的索引
	notify uint32

	lock mutex
	head *sudog//head 和 tail 分别指向的链表的头和尾
	tail *sudog
}
```

##### wait方法

wait方法会将当前 Goroutine 陷入休眠状态，

```go
func (c *Cond) Wait() {
	c.checker.check()
	t := runtime_notifyListAdd(&c.notify) // 等待计数器加一并解
	c.L.Unlock()
	runtime_notifyListWait(&c.notify, t) // 会获取当前 Goroutine 并将它追加到 Goroutine 通知链表的最末端
	c.L.Lock()
}

func notifyListAdd(l *notifyList) uint32 {
	return atomic.Xadd(&l.wait, 1) - 1
}
func notifyListWait(l *notifyList, t uint32) {
	s := acquireSudog()
	s.g = getg()
	s.ticket = t
	if l.tail == nil {
		l.head = s
	} else {
		l.tail.next = s
	}
	l.tail = s
	goparkunlock(&l.lock, waitReasonSyncCondWait, traceEvGoBlockCond, 3)
	releaseSudog(s)
}

小结 #
```

##### 唤醒

 sync.Cond.Signal 或者 sync.Cond.Broadcast 唤醒一个或者全部的 Goroutine。Signal 唤醒队列最前面的 Goroutine，Broadcast 方法会唤醒队列中全部的 Goroutine
