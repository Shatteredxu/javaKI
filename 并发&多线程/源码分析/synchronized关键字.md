### synchronized关键字

https://blog.csdn.net/javazejian/article/details/72828483

#### 锁对象

* 修饰实例方法，作用于当前实例加锁，进入同步代码前要获得当前实例的锁

  对于每一个new对象创建的实例，synchronized修饰的方法为实例方法，属于每个实例。

* 修饰静态方法，作用于当前类对象加锁，进入同步代码前要获得当前类对象的锁

  由static修饰的方法，锁的是类对象

* 修饰代码块，指定加锁对象，对给定对象加锁，进入同步代码库前要获得给定对象的锁。

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

![img](assets/20170603172215966)

上图中的重量级锁也就是synchronized的对象锁，锁标识位为10，其中指针指向的是monitor对象（也称为管程或监视器锁）的起始地址。每个对象都存在着一个 monitor 与之关联，当一个 monitor 被某个线程持有后，它便处于锁定状态。



同步语句块的实现使用的是**monitorenter 和 monitorexit 指令**，其中monitorenter指令指向同步代码块的开始位置，monitorexit指令则指明同步代码块的结束位置，当执行monitorenter指令时，当前线程将试图获取 objectref(即对象锁) 所对应的 monitor 的持有权，当 objectref 的 monitor 的进入计数器为 0，那线程可以成功取得 monitor，并将计数器值设置为 1，取锁成功。如果当前线程已经拥有 objectref 的 monitor 的持有权，那它可以重入这个 monitor (关于重入性稍后会分析)，重入时计数器的值也会加 1。倘若其他线程已经拥有 objectref 的 monitor 的所有权，那当前线程将被阻塞，直到正在执行线程执行完毕，即monitorexit指令被执行，执行线程将释放 monitor(锁)并设置计数器值为0 ，其他线程将有机会持有 monitor 。值得注意的是编译器将会确保无论方法通过何种方式完成，方法中调用过的每条 monitorenter 指令都有执行其对应 monitorexit 指令，而无论这个方法是正常结束还是异常结束。为了保证在方法异常完成时 monitorenter 和 monitorexit 指令依然可以正确配对执行，**编译器会自动产生一个异常处理器**，这个异常处理器声明可处理所有的异常，它的目的就是用来执行 monitorexit 指令。从字节码中也可以看出多了一个monitorexit指令，它就是异常结束时被执行的释放monitor 的指令

### synchronized的优化

无锁–>偏向锁—>轻量级锁—>重量级锁

##### 偏向锁

可就是可重入锁

##### 轻量级锁

##### 自旋锁

1. synchronized(Integer)

   

2. synchronized(null)

报空指针异常