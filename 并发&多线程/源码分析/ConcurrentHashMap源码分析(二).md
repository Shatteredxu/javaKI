## ConcurrentHashMap源码分析(二)

https://blog.csdn.net/zzu_seu/article/details/106698150

https://www.jianshu.com/p/487d00afe6ca

>   JDK1.8的CHM考点：
>
>   1.   初始化长度，扩容阈值延迟初始化，初始化容量为比传入的大的2的幂次方
>
>   2.   将1.7Segment改为Node，
>
>   3.   sizeCtl的值表示当前数组的状态
>
>   4.   put方法，先初始化，在判断Ctl值，如果不为-1，则尝试修改为-1，初始化Node数组，如果发现has值为Moved，则帮助扩容（多线程），如果都没有，则执行正常的添加操作（需要将桶锁住），添加到链表或者红黑树中，之后再判断binCount是否大于8, 是则进行转换
>
>   5.   扩容细节：sizeCtl扩容时代表当前扩容的线程//  transferIndex：**扩容索引，表示已经分配给扩容线程的table数组索引位置**,//
>
>        ForwardingNode节点:标记作用，表示其他线程正在扩容，并且此节点已经扩容完毕; 关联了nextTable,扩容期间可以通过find方法，访问已经迁移到了nextTable中的数据,何时扩容，// 迁移桶中的元素时会分成两份和hashmap一样 //线程2访问到了ForwardingNode节点，执行的put或remove等写操作，那么就会先帮其扩容,执行的是get等读方法，则会调用ForwardingNode的find方法，去nextTable里面查找相关元素。
>
>        如果准备加入扩容的线程，发现以下情况，放弃扩容，直接返回。
>
>        *   发现transferIndex=0,即**所有node均已分配**
>        *   发现扩容线程已经达到**最大扩容线程数**
>
>        tryPresize方法协调多个线程如何调用transfer方法进行hash桶的迁移
>        
>   6.   集合长度计算

### 一、jdk1.8容器初始化

##### jdk1.7的优势

（1）减少Hash冲突，避免一个槽里有太多元素。

（2）提高读和写的并发度。段与段之间相互独立。

（3）提供扩容的并发度。扩容的时候，不是整个ConcurrentHashMap 一起扩容，而是每个Segment独立扩容。

##### jdk1.8针对的改进

（1）使用红黑树，当一个槽里有很多元素时，其查询和更新速度会比链表快很多，Hash冲突的问题由此得到较好的解决。

（2）加锁的粒度，并非整个ConcurrentHashMap，而是对每个头节点分别加锁，即并发度，就是Node数组的长度，初始长度为16，和在JDK 7中初始Segment的个数相同。

（3）并发扩容，这是难度最大的。在JDK 7中，一旦Segment的个数在初始化的时候确立，不能再更改，并发度被固定。之后只是在每个Segment内部扩容，这意味着每个Segment独立扩容，互不影响，不存在并发扩容的问题。但在JDK 8中，相当于只有1个Segment，当一个线程要扩容Node数组的时候，其他线程还要读写，因此处理过程很复杂，后面会详细分析。



#### 1、源码分析

> 在jdk8的ConcurrentHashMap中一共有5个构造方法，这四个构造方法中都没有对内部的数组做初始化， 只是对一些变量的初始值做了处理
>
> jdk8的ConcurrentHashMap的数组初始化是在第一次添加元素时完成

```java
//没有维护任何变量的操作，如果调用该方法，数组长度默认是16
public ConcurrentHashMap() {
}
```

```java
//传递进来一个初始容量，ConcurrentHashMap会基于这个值计算一个比这个值大的2的幂次方数作为初始容量
public ConcurrentHashMap(int initialCapacity) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException();
    int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?//cap就是Node数组的长度,保持为2的整数次方
               MAXIMUM_CAPACITY :
               //tableSizeFor 根据传入的值，计算合适的数组长度，就是1.5倍的初始容量+1，再往上取最接近的2的整数次方，作为数组长度cap的初始值。
               tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
    this.sizeCtl = cap;//用于控制在初始化或者并发扩容时候的线程数，只不过其初始值设置成c//
}
```

> 注意，调用这个方法，得到的初始容量和我们之前讲的HashMap以及jdk7的ConcurrentHashMap不同，即使你传递的是一个2的幂次方数，该方法计算出来的初始容量依然是比这个值大的2的幂次方数

```java
//调用四个参数的构造
public ConcurrentHashMap(int initialCapacity, float loadFactor) {
    this(initialCapacity, loadFactor, 1);
}
```

```java
//计算一个大于或者等于给定的容量值，该值是2的幂次方数作为初始容量
public ConcurrentHashMap(int initialCapacity,
                         float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    if (initialCapacity < concurrencyLevel)   // Use at least as many bins
        initialCapacity = concurrencyLevel;   // as estimated threads
    long size = (long)(1.0 + (long)initialCapacity / loadFactor);
    int cap = (size >= (long)MAXIMUM_CAPACITY) ?
        MAXIMUM_CAPACITY : tableSizeFor((int)size);
    this.sizeCtl = cap;
}
```

```java
//基于一个Map集合，构建一个ConcurrentHashMap
//初始容量为16
public ConcurrentHashMap(Map<? extends K, ? extends V> m) {
    this.sizeCtl = DEFAULT_CAPACITY;
    putAll(m);
}
```

#### 2、`sizeCtl`含义解释

> **注意：以上这些构造方法中，都涉及到一个变量`sizeCtl`，这个变量是一个非常重要的变量，而且具有非常丰富的含义，它的值不同，对应的含义也不一样，这里我们先对这个变量不同的值的含义做一下说明，后续源码分析过程中，进一步解释**
>
> `sizeCtl`为0，代表数组未初始化， 且数组的初始容量为16
>
> `sizeCtl`为正数，如果数组未初始化，那么其记录的是数组的初始容量，如果数组已经初始化，那么其记录的是数组的扩容阈值
>
> `sizeCtl`为-1，表示数组正在进行初始化
>
> `sizeCtl`小于0，并且不是-1，表示数组正在扩容， -(1+n)，表示此时有n个线程正在共同完成数组的扩容操作

### 二、jdk1.8添加安全

#### 1、源码分析

##### 1.1、添加元素put/putVal方法

```java
public V put(K key, V value) {
    return putVal(key, value, false);
}
```

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    //如果有空值或者空键，直接抛异常
    if (key == null || value == null) throw new NullPointerException();
    //基于key计算hash值，并进行一定的扰动
    int hash = spread(key.hashCode());
    //记录某个桶上元素的个数，如果超过8个，会转成红黑树
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        //如果数组还未初始化，先对数组进行初始化
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
	    //如果hash计算得到的桶位置没有元素，利用cas将元素添加
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            //cas+自旋（和外侧的for构成自旋循环），保证元素添加安全
            if (casTabAt(tab, i, null,////如果这个位置没有元素的话，则通过cas的方式尝试添加，注意这个时候是没有加锁的
                         new Node<K,V>(hash, key, value, null)))////创建一个Node添加到数组中区，null表示的是下一个节点为空
                break;                   // no lock when adding to empty bin
        }
        //如果hash计算得到的桶位置元素的hash值为MOVED，证明正在扩容，那么协助扩容
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);//如何多线程扩容
        else {
            //hash计算的桶位置元素不为空，且当前没有处于扩容操作，进行元素添加
            V oldVal = null;
            //对当前桶进行加锁，保证线程安全，执行元素添加操作
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    //普通链表节点
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    //树节点，将元素添加到红黑树中
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {
                //链表长度大于/等于8，将链表转成红黑树
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                //如果是重复键，直接将旧值返回
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    //添加的是新元素，维护集合长度，并判断是否要进行扩容操作
    addCount(1L, binCount);
    return null;
}
```

> **通过以上源码，我们可以看到，当需要添加元素时，会针对当前元素所对应的桶位进行加锁操作，这样一方面保证元素添加时，多线程的安全，同时对某个桶位加锁不会影响其他桶位的操作，进一步提升多线程的并发效率**

##### 1.2、数组初始化，initTable方法

```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    //cas+自旋，保证线程安全，对数组进行初始化操作
    while ((tab = table) == null || tab.length == 0) {
        //如果sizeCtl的值（-1）小于0，说明此时正在初始化， 让出cpu；
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
        //cas修改sizeCtl的值为-1，修改成功，进行数组初始化，失败，继续自旋
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
          //如果修改成功，则可以开始初始化，初始化后，其他自旋线程检测到数组不为null，则退出
            try {
                if ((tab = table) == null || tab.length == 0) {
                    //sizeCtl为0，取默认长度16，否则去sizeCtl的值
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    //基于初始长度，构建数组对象
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    //计算扩容阈值，并赋值给sc
                    sc = n - (n >>> 2);//sc=0.75,
                }
            } finally {
                //将扩容阈值，赋值给sizeCtl//初始化后，sizeCtl长度为数组长度的3/4，即下次扩容阈值
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```

构造函数里只计算了数组的初始大小，并没有对数组进行初始化。当多个线程都往里面放入元素的时候，再进行初始化。这就存在一个问题：多个线程重复初始化，如果某个线程成功地把sizeCtl 设置为-1，它就拥有了初始化的权利，进入初始化的代码模块，等到初始化完成，再把sizeCtl设置回去

#### 2、图解

##### 2.1、put加锁图解

![](assets/Snipaste_2020-05-16_12-04-08.jpg)



### 三、jdk1.8扩容

什么时候会进行扩容？

1、如果新增节点之后，所在链表的元素个数达到了阈值8，则会调用treeifyBin方法把链表转换成红黑树，不过在结构转换之前，会对数组长度进行判断，实现如下：如果数组长度n小于阈值MIN_TREEIFY_CAPACITY，默认是64，则会调用tryPresize方法把数组长度扩大到原来的两倍，并触发transfer方法，重新调整节点的位置

2、调用put方法新增节点时，在结尾会调用addCount方法记录元素个数，并检查是否需要进行扩容，当数组元素个数达到阈值时，会触发transfer方法，重新调整节点的位置

#### 1、源码分析

```java
 private final void addCount(long x, int check) {
        CounterCell[] as; long b, s;
        if ((as = counterCells) != null ||
            !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
            CounterCell a; long v; int m;
            boolean uncontended = true;
            if (as == null || (m = as.length - 1) < 0 ||
                (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
                !(uncontended =
                  U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
                fullAddCount(x, uncontended);
                return;
            }
            if (check <= 1)
                return;
            s = sumCount();
        }
        if (check >= 0) {
            Node<K,V>[] tab, nt; int n, sc;
            while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                   (n = tab.length) < MAXIMUM_CAPACITY) {
                int rs = resizeStamp(n);
              //sc<0表示已经有线程在进行扩容工作或者在进行初始化
                if (sc < 0) {
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;
                  //添加一个线程进行扩容
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                        transfer(tab, nt);
                }
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))
                    transfer(tab, null);
                s = sumCount();
            }
        }
    }
```

```java
//addCount中调用，helpTransfer帮助扩容，putAll批量插入时判断
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    //如果是多cpu，那么每个线程划分任务，最小任务量是16个桶位的迁移
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)//计算一个步长，表示一个线程处理的数组长度，用来控制对CPU的使用
        stride = MIN_TRANSFER_STRIDE; // 默认最小16，如果一个数组的长度只有16，那只有一个线程会对其进行扩容的复制移动操作
    //如果是扩容线程，此时新数组为null
    if (nextTab == null) {  //只有第一次的时候会扩容两倍，如果是帮助扩容则不需要初始化2倍数组
        try {
            @SuppressWarnings("unchecked")
            //两倍扩容创建新数组
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];//扩容两倍
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;//nextTab为新建的扩容后的数组
        //记录线程开始迁移的桶位，从后往前迁移，n为旧的hashmap的长度
        transferIndex = n;
    }
    //记录新数组的末尾

    int nextn = nextTab.length;
    	//创建一个fwd节点，这个是用来控制并发的，当一个节点为空或已经被转移之后，就设置为fwd节点
    //已经迁移的桶位，会用这个节点占位（这个节点的hash值为-1--MOVED）
  
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    boolean advance = true;
    boolean finishing = false; // to ensure sweep before committing nextTab
  //此循环的作用是确定当前线程要迁移的桶的范围或通过更新i的值确定当前范围内下一个要处理的节点。
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
      //advance是遍历的确认标志，是否再往前进行遍历
        while (advance) {//下面有三个分支，会将advance置为false，都不满足的时候就会一直自旋获取迁移任务
            int nextIndex, nextBound;
            //i记录当前正在迁移桶位的索引值
            //bound记录下一次任务迁移的开始桶位
            
            //--i >= bound 成立表示当前线程分配的迁移任务还没有完成
            if (--i >= bound || finishing)//i==bound
                advance = false;
            //没有元素需要迁移 -- 后续会去将扩容线程数减1，并判断扩容是否完成
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            //计算下一次任务迁移的开始桶位，并将这个值赋值给transferIndex
          	//cas无锁算法设置 transferIndex = transferIndex - stride
            else if (U.compareAndSwapInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
        //如果没有更多的需要迁移的桶位，就进入该if
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            //扩容结束后，保存新数组，并重新计算扩容阈值，赋值给sizeCtl
            if (finishing) {
                nextTable = null;
                table = nextTab;
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
		   //扩容任务线程数减1
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                //判断当前所有扩容任务线程是否都执行完成
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                //所有扩容线程都执行完，标识结束
                finishing = advance = true;
                i = n; // recheck before commit
            }
        }
        //当前迁移的桶位没有元素，直接在该位置添加一个fwd节点
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
        //当前节点已经被迁移
        else if ((fh = f.hash) == MOVED)
            advance = true; // already processed
        else {
            //当前节点需要迁移，加锁迁移，保证多线程安全
            //此处迁移逻辑和jdk7的ConcurrentHashMap相同，不再赘述
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;
                    if (fh >= 0) {
                        int runBit = fh & n;
                        Node<K,V> lastRun = f;
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                       //低位链表放在i处
                        setTabAt(nextTab, i, ln);
                       //高位链表放在i+n处
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                  //同样也是使用高位和低位两条链表进行迁移
      					//使用for循环以链表方式遍历整棵红黑树，使用尾插法拼接 ln 和 hn 链表
                    else if (f instanceof TreeBin) {
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                            (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                            (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                }
            }
        }
    }
}
```

```java
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
        Node<K,V>[] nextTab; int sc;
        if (tab != null && (f instanceof ForwardingNode) &&
            (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
            int rs = resizeStamp(tab.length);
            while (nextTab == nextTable && table == tab &&
                   (sc = sizeCtl) < 0) {
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || transferIndex <= 0)
                    break;
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                    transfer(tab, nextTab);
                    break;
                }
            }
            return nextTab;
        }
        return table;
    }
```



#### 2、图解

![](assets/微信截图_20200516213316.png)



### 四、jdk1.8多线程扩容效率改进

> 多线程协助扩容的操作会在两个地方被触发：
>
> ① 当添加元素时，发现添加的元素对用的桶位为fwd节点，就会先去协助扩容，然后再添加元素
>
> ② 当添加完元素后，判断当前元素个数达到了扩容阈值，此时发现sizeCtl的值小于0，并且新数组不为空，这个时候，会去协助扩容

#### 1、源码分析

##### 1.1、元素未添加，先协助扩容，扩容完后再添加元素

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();//1.整个数组初始化
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {//第i个元素初始化
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        //发现此处为fwd节点，协助扩容，扩容结束后，再循环回来添加元素
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);//协助扩容
        
        //省略代码
```

```java
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    if (tab != null && (f instanceof ForwardingNode) &&
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        int rs = resizeStamp(tab.length);
        while (nextTab == nextTable && table == tab &&
               (sc = sizeCtl) < 0) {
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                //扩容，传递一个不是null的nextTab
                transfer(tab, nextTab);
                break;
            }
        }
        return nextTab;
    }
    return table;
}
```

##### 1.2、先添加元素，再协助扩容

```java
private final void addCount(long x, int check) {
    //省略代码
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
  	    //元素个数达到扩容阈值
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
               (n = tab.length) < MAXIMUM_CAPACITY) {
            int rs = resizeStamp(n);
            //sizeCtl小于0，说明正在执行扩容，那么协助扩容
            if (sc < 0) {
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
            s = sumCount();
        }
    }
}
```



> 注意：扩容的代码都在transfer方法中，这里不再赘述

#### 2、图解

![](assets/微信截图_20200516215323.png)



### 五、集合长度的累计方式

#### 1、源码分析

##### 1.1、addCount方法

> **① CounterCell数组不为空，优先利用数组中的CounterCell记录数量**
>
> **② 如果数组为空，尝试对baseCount进行累加，失败后，会执行fullAddCount逻辑**
>
> **③ 如果是添加元素操作，会继续判断是否需要扩容**

```java
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    //当CounterCell数组不为空，则优先利用数组中的CounterCell记录数量
    //或者当baseCount的累加操作失败，会利用数组中的CounterCell记录数量
    if ((as = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        CounterCell a; long v; int m;
        //标识是否有多线程竞争
        boolean uncontended = true;
        //当as数组为空
        //或者当as长度为0
        //或者当前线程对应的as数组桶位的元素为空
        //或者当前线程对应的as数组桶位不为空，但是累加失败
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended =
              U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
            //以上任何一种情况成立，都会进入该方法，传入的uncontended是false
            fullAddCount(x, uncontended);
            return;
        }
        if (check <= 1)
            return;
        //计算元素个数
        s = sumCount();
    }
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        //当元素个数达到扩容阈值
        //并且数组不为空
        //并且数组长度小于限定的最大值
        //满足以上所有条件，执行扩容
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
               (n = tab.length) < MAXIMUM_CAPACITY) {
            //这个是一个很大的正数
            int rs = resizeStamp(n);
            //sc小于0，说明有线程正在扩容，那么会协助扩容
            if (sc < 0) {
                //扩容结束或者扩容线程数达到最大值或者扩容后的数组为null或者没有更多的桶位需要转移，结束操作
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                //扩容线程加1，成功后，进行协助扩容操作
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    //协助扩容，newTable不为null
                    transfer(tab, nt);
            }
            //没有其他线程在进行扩容，达到扩容阈值后，给sizeCtl赋了一个很大的负数
            //1+1=2 --》 代表此时有一个线程在扩容
            
            //rs << RESIZE_STAMP_SHIFT)是一个很大的负数
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                //扩容，newTable为null
                transfer(tab, null);
            s = sumCount();
        }
    }
}
```

##### 1.2、fullAddCount方法

> **① 当CounterCell数组不为空，优先对CounterCell数组中的CounterCell的value累加**
>
> **② 当CounterCell数组为空，会去创建CounterCell数组，默认长度为2，并对数组中的CounterCell的value累加**
>
> **③ 当数组为空，并且此时有别的线程正在创建数组，那么尝试对baseCount做累加，成功即返回，否则自旋**

```java
private final void fullAddCount(long x, boolean wasUncontended) {
    int h;
    //获取当前线程的hash值
    if ((h = ThreadLocalRandom.getProbe()) == 0) {
        ThreadLocalRandom.localInit();      // force initialization
        h = ThreadLocalRandom.getProbe();
        wasUncontended = true;
    }
    //标识是否有冲突，如果最后一个桶不是null，那么为true
    boolean collide = false;                // True if last slot nonempty
    for (;;) {
        CounterCell[] as; CounterCell a; int n; long v;
        //数组不为空，优先对数组中CouterCell的value累加
        if ((as = counterCells) != null && (n = as.length) > 0) {
            //线程对应的桶位为null
            if ((a = as[(n - 1) & h]) == null) {
                if (cellsBusy == 0) {            // Try to attach new Cell
                    //创建CounterCell对象
                    CounterCell r = new CounterCell(x); // Optimistic create
                    //利用CAS修改cellBusy状态为1，成功则将刚才创建的CounterCell对象放入数组中
                    if (cellsBusy == 0 &&
                        U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                        boolean created = false;
                        try {               // Recheck under lock
                            CounterCell[] rs; int m, j;
                            //桶位为空， 将CounterCell对象放入数组
                            if ((rs = counterCells) != null &&
                                (m = rs.length) > 0 &&
                                rs[j = (m - 1) & h] == null) {
                                rs[j] = r;
                                //表示放入成功
                                created = true;
                            }
                        } finally {
                            cellsBusy = 0;
                        }
                        if (created) //成功退出循环
                            break;
                        //桶位已经被别的线程放置了已给CounterCell对象，继续循环
                        continue;           // Slot is now non-empty
                    }
                }
                collide = false;
            }
            //桶位不为空，重新计算线程hash值，然后继续循环
            else if (!wasUncontended)       // CAS already known to fail
                wasUncontended = true;      // Continue after rehash
            //重新计算了hash值后，对应的桶位依然不为空，对value累加
            //成功则结束循环
            //失败则继续下面判断
            else if (U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))
                break;
            //数组被别的线程改变了，或者数组长度超过了可用cpu大小，重新计算线程hash值，否则继续下一个判断
            else if (counterCells != as || n >= NCPU)
                collide = false;            // At max size or stale
            //当没有冲突，修改为有冲突，并重新计算线程hash，继续循环
            else if (!collide)
                collide = true;
            //如果CounterCell的数组长度没有超过cpu核数，对数组进行两倍扩容
            //并继续循环
            else if (cellsBusy == 0 &&
                     U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                try {
                    if (counterCells == as) {// Expand table unless stale
                        CounterCell[] rs = new CounterCell[n << 1];
                        for (int i = 0; i < n; ++i)
                            rs[i] = as[i];
                        counterCells = rs;
                    }
                } finally {
                    cellsBusy = 0;
                }
                collide = false;
                continue;                   // Retry with expanded table
            }
            h = ThreadLocalRandom.advanceProbe(h);
        }
        //CounterCell数组为空，并且没有线程在创建数组，修改标记，并创建数组
        else if (cellsBusy == 0 && counterCells == as &&
                 U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
            boolean init = false;
            try {                           // Initialize table
                if (counterCells == as) {
                    CounterCell[] rs = new CounterCell[2];
                    rs[h & 1] = new CounterCell(x);
                    counterCells = rs;
                    init = true;
                }
            } finally {
                cellsBusy = 0;
            }
            if (init)
                break;
        }
        //数组为空，并且有别的线程在创建数组，那么尝试对baseCount做累加，成功就退出循环，失败就继续循环
        else if (U.compareAndSwapLong(this, BASECOUNT, v = baseCount, v + x))
            break;                          // Fall back on using base
    }
}
```

#### 2、图解

> fullAddCount方法中，当as数组不为空的逻辑图解

![](assets/微信截图_20200516185407.png)



### 六、jdk1.8集合长度获取

#### 1、源码分析

##### 1.1、size方法

```java
public int size() {
    long n = sumCount();
    return ((n < 0L) ? 0 :
            (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
            (int)n);
}
```

##### 1.2、sumCount方法

```java
final long sumCount() {
    CounterCell[] as = counterCells; CounterCell a;
    //获取baseCount的值
    long sum = baseCount;
    if (as != null) {
        //遍历CounterCell数组，累加每一个CounterCell的value值
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```

> 注意：这个方法并不是线程安全的



