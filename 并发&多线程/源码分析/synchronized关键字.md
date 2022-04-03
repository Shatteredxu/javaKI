### synchronized关键字

https://blog.csdn.net/javazejian/article/details/72828483

#### 锁对象

* **修饰实例方法**，作用于当前实例加锁，进入同步代码前要获得当前实例的锁, 对于每一个new对象创建的实例，synchronized修饰的方法为实例方法，属于每个实例。

* **修饰静态方法**，作用于当前类对象加锁，进入同步代码前要获得当前类对象的锁, 由static修饰的方法，**锁的是类对象**

* **修饰代码块**，指定加锁对象，对给定对象加锁，synchronized(this|object)表示<u>进入同步代码库前要获得给定对象的锁</u>。

  ```
   synchronized(this){
     for(int j=0;j<1000000;j++){
        i++;
     }
  }
  ```

  ### synchronized底层语义原理

  在JVM中，对象在内存中的布局分为三块区域：**对象头**、**实例数据**和**对齐填充**

  对象头由由Mark Word 和 Class Metadata Address 组成，Mark Word在默认情况下存储着对象的HashCode、分代年龄、锁标记位等

<img src="assets/image-20220228112450934.png" alt="image-20220228112450934" style="zoom:80%;" />

上图中的重量级锁也就是synchronized的对象锁，锁标识位为10，其中<u>指针指向的是monitor对象</u>（也称为管程或监视器锁）<u>的起始地址</u>。每个对象都存在着一个 monitor 与之关联，当一个 monitor 被某个线程持有后，它便处于锁定状态。<u>monitor是由ObjectMonitor实现的</u>，ObjectMonitor中有两个队列，_WaitSet 和 _EntryList，用来保存ObjectWaiter对象列表( 每个等待锁的线程都会被封装成ObjectWaiter对象)，_owner指向持有ObjectMonitor对象的线程，当多个线程同时访问一段同步代码时，首先会进入 _EntryList 集合，<u>当线程获取到对象的monitor 后进入 _Owner 区域并把monitor中的owner变量设置为当前线程同时monitor中的计数器count加1</u>，若线程调用 wait() 方法，将释放当前持有的monitor，owner变量恢复为null，count自减1，同时该线程进入 WaitSe t集合中等待被唤醒。若当前线程执行完毕也将释放monitor(锁)并复位变量的值，以便其他线程进入获取monitor(锁)



同步语句块的实现使用的是**monitorenter 和 monitorexit 指令**，其中monitorenter指令指向同步代码块的开始位置，monitorexit指令则指明同步代码块的结束位置，当执行monitorenter指令时，当前线程将试图获取 objectref(即对象锁) 所对应的 monitor 的持有权，当 objectref 的 monitor 的进入计数器为 0，那线程可以成功取得 monitor，并将计数器值设置为 1，取锁成功。如果当前线程已经拥有 objectref 的 monitor 的持有权，那它可以重入这个 monitor ，重入时计数器的值也会加 1。倘若其他线程已经拥有 objectref 的 monitor 的所有权，那当前线程将被阻塞，直到正在执行线程执行完毕，即monitorexit指令被执行，执行线程将释放 monitor(锁)并设置计数器值为0 ，其他线程将有机会持有 monitor 。值得注意的是编译器将会确保无论方法通过何种方式完成，方法中调用过的每条 monitorenter 指令都有执行其对应 monitorexit 指令，而无论这个方法是正常结束还是异常结束。为了保证在方法异常完成时 monitorenter 和 monitorexit 指令依然可以正确配对执行，**编译器会自动产生一个异常处理器**，这个异常处理器声明可处理所有的异常，它的目的就是用来执行 monitorexit 指令。从字节码中也可以看出多了一个monitorexit指令，它就是异常结束时被执行的释放monitor 的指令

### synchronized的优化

无锁–>偏向锁—>轻量级锁—>重量级锁

##### 偏向锁

可就是可重入锁

##### 轻量级锁

##### 自旋锁

1. synchronized(Integer)

2. synchronized(null)

报空指针异常

### synchronized 关键字和volatile 关键字

**volatile ：**防止指令重排，保证可见性。

synchronized 关键字和 volatile 关键字是两个互补的存在，而不是对立的存在！

1.   volatile 关键字是线程同步的轻量级实现，所以volatile 性能肯定比synchronized关键字要好。但是volatile 关键字只能用于变量而 synchronized 关键字可以修饰方法以及代码块。
2.   volatile 关键字能保证数据的可见性，但不能保证数据的原子性。synchronized 关键字两者都能保证。
3.   volatile关键字主要用于解决变量在多个线程之间的可见性，而 synchronized 关键字解决的是多个线程之间访问资源的同步性。
4.   多线程访问volatile关键字不会发生阻塞，而synchronized关键字可能会发生阻塞。

### synchronized和Lock

##### 相同点：

- synchronized 和 Lock 都是用来保护资源线程安全的。
- 都可以保证可见性
- synchronized 和 ReentrantLock 都拥有可重入的特点

##### 不同点：

1. synchronized 关键字可以加在方法上，不需要指定锁对象（此时的锁对象为 this），也可以新建一个同步代码块并且自定义 monitor 锁对象；而 Lock 接口必须显示用 Lock 锁对象开始加锁 lock() 和解锁 unlock()，并且一般会在 finally 块中确保用 unlock() 来解锁，以防发生死锁。

   与 Lock 显式的加锁和解锁不同的是 synchronized 的加解锁是隐式的，尤其是抛异常的时候也能保证释放锁，但是 Java 代码中并没有相关的体现。

2. 如果有多把 Lock 锁，Lock 可以不完全按照加锁的反序解锁，比如我们可以先获取 Lock1 锁，再获取 Lock2 锁，解锁时则先解锁 Lock1，再解锁 Lock2，加解锁有一定的灵活度，但是synchronized不能，必须要按照顺序去加解锁。

3. synchronized 锁不够灵活；不能手动的去释放锁资源，可能会出现锁很久得不到释放的情况，而Lock锁如果使用的是 lockInterruptibly 方法，那么如果觉得等待的时间太长了不想再继续等待，可以中断退出，也可以用 tryLock() 等方法尝试获取锁，如果获取不到锁也可以做别的事，更加灵活。

4. synchronized 锁只能同时被一个线程拥有，但是 Lock 锁没有这个限制,lock锁下面的读写锁可以被多给线程所拥有

5. lock可以设置公平/非公平

   

##### 如何选择

1. 如果能不用最好既不使用 Lock 也不使用 synchronized。因为在许多情况下你可以使用 java.util.concurrent 包中的机制，它会为你处理所有的加锁和解锁操作，也就是推荐**优先使用工具类来加解锁**。
2. 如果 synchronized 关键字适合你的程序， 那么请尽量使用它，这样可以减少编写代码的数量，减少出错的概率。因为一旦忘记在 finally 里 unlock，代码可能会出很大的问题，而**使用 synchronized 更安全**。
3. 如果特别**需要 Lock 的特殊功能**，比如尝试获取锁、可中断、超时功能等，才使用 Lock。