## HashMap源码

>   HashMap知识点：
>
>          1. 初始化默认长度1，负载因子
>          2. 构造器：根据传入值初始化cap，使用tableSizeFor返回大于cap的2的幂次方
>          3. hash方法的计算(key.hashcode ^ (key.hashcode>>>16))融合高位低位特征
>          4. getNode函数可以知道节点有没有出现，但是通过get不能知道，因为hashmap允许key，value都为null
>          5. put源码（1. 懒加载，没有初始化则resize初始化2. 通过hashcode得到table[i]位置元素，判断有没有等于key的值，又分为链表和红黑树的判断，如果存在元素,则更新value，不存在则执行链表或者红黑树的插入，链表插入后需要判断是否扩容，如果需要则转为红黑树）//如果table数组的长度<64,此时进行扩容操作//如果table数组的长度>64，此时进行链表转红黑树结构的操作
>          6. resize分为两个链表进行扩容，**为什么分为两个链表，怎么推导**？

[hashMap源码分析](https://blog.csdn.net/liewen_/article/details/82940272?utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromMachineLearnPai2%7Edefault-1.control&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromMachineLearnPai2%7Edefault-1.control)

JDK1.8使用数组+链表+红黑树

JDK1.7使用数组+链表

### 1.属性

继承自AbstractMap，实现Map<K,V>, Cloneable, Serializable，AbstractMap也实现了Map接口，这其实是个多余的操作

```java
DEFAULT_INITIAL_CAPACITY= 1 << 4; //默认长度 16
MAXIMUM_CAPACITY  = 1 << 30//最大容量
DEFAULT_LOAD_FACTOR= 0.75f  //负载因子 
//当某一个桶中链表的长度>=8时，链表结构会转换成红黑树结构（其实还要求桶的中数量>=64,后面会提到）
static final int TREEIFY_THRESHOLD = 8;
//当红黑树中的节点数量<=6时，红黑树结构会转变为链表结构
static final int UNTREEIFY_THRESHOLD = 6;
//上面提到的：当Node数组容量>=64的前提下，如果某一个桶中链表长度>=8，则会将链表结构转换成  //红黑树结构
    static final int MIN_TREEIFY_CAPACITY = 64
Node<K,V>[] table 存储元素
int threshold;数组扩容的边界值 大于的话就要开始扩容了
```

### 2.构造器

```java
//有参构造器，一般不太需要，HashMap是“懒加载”，在构造器中值保留了相关保留的值，并没有初始化table<Node>数组，当我们向map中put第一个元素的时候，map才会进行初始化！
public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)//如果传了超大数，就赋值为最大值
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
  			//通过tableSizeFor返回大于initialCapacity的2的幂次方值
        this.threshold = tableSizeFor(initialCapacity);
    }

HashMap(int initialCapacity)

```

### hash方法

```java
//Node是HashMap的内部类对标1.7的Entry
static class Node<K,V> implements Map.Entry<K,V> {
        /* 新的hash = key.hashcode ^ (key.hashcode>>>16)将hashcode无符号右移得到高位的数字，再将高位数字异或进hashcode中，这样就融合了整个hashcode的特征，来减少hash冲突。当我们往map中put(k,v)时，这个k,v键值对会被封装为Node，那么这个Node放在Node数组的哪个位置呢：index=hash&(n-1),n为Node数组的长度。如果直接使用hashCode&(n-1)来计算index，此时hashCode的高位随机特性完全没有用到，因为n相对于 hashcode的值很小。基于这一点，把hashcode 高16位的值通过异或混合到hashCode的低16位，由此来增强hashCode低16位的随机性 */
        final int hash; 
        final K key;//保存map中的key
        V value;//保存map中的value
        Node<K,V> next;//单向链表
 
        Node(int hash, K key, V value, Node<K,V> next) {//构造器
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
```

hash和tableSizeFor源码

```java
//
static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);//加入高16位元素，与原始的低16位进行异或
    }
//获得传入的初始容量值大于等于cap并且是2的整数次幂的所有数中最小的那个，即返回一个最接近cap(>=cap)，并且是2的整数次幂的那个数
 static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;有1个1
        n |= n >>> 2;有2个1
        n |= n >>> 4;有4个1
        n |= n >>> 8;有8个1
        n |= n >>> 16;有16个1，最后有32个1
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;//n+1代表将所有1去掉得到稍大的那个2的整数幂的值
    }
```

### get方法

1. 通过hashcode得到在Hash数组中的位置，查看是否为空
2. 不为空，如果table[index]处节点的key就是要找的key则直接返回该节点；
3. 再分红黑树的搜索和链表的搜索

```java

//入口,返回对应的value
public V get(Object key) {
        Node<K,V> e;
        /**
        hash：这个函数上面分析过了。返回key混淆后的hashCode
        注意getNode返回的类型是Node：当返回值为null时表示map中没有对应的key，注意区分value为null：         如果key对应的value为null的话，体现在getNode的返回值e.value为null，此时返回值也是
        null，也就是HashMap的get函数不能判断map中是否有对应的key：get返回值为null时，可能不包 
        含该key，也可能该key的value为null！那么如何判断map中是否包含某个key呢？见下面contains 
        **/
        //函数分析
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }
    public boolean containsKey(Object key) {
        //注意与get函数区分，我们往map中put的所有的<key,value>都被封装在Node中，如果Node都不存在
        //显然一定不包含对应的key
        return getNode(hash(key), key) != null;
    }
 
  //下面分析getNode函数
  /**
     * Implements Map.get and related methods
     *
     * @param hash hash for key //下面简称：key的hash值
     * @param key the key
     * @return the node, or null if none
     */
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        //(n-1)&hash：当前key可能在的桶索引，put操作时也是将Node存放在index=(n-1)&hash位置
        //主要逻辑：如果table[index]处节点的key就是要找的key则直接返回该节点；
        //否则：如果在table[index]位置进行搜索，搜索是否存在目标key的Node：这里的搜索又分两种：
        //链表搜索和红黑树搜索，具体红黑树的查找就不展开了，红黑树是一种弱平衡(相对于AVL)BST，
        //红黑树查找、删除、插入等操作都能够保证在O(lon(n))时间复杂度内完成，红黑树原理不在本文
        //范围内，但是大家要知道红黑树的各种操作是可以实现的，简单点可以把红黑树理解为BST，
        //BST的查找、插入、删除等操作的实现在之前的博客中有java实现代码，实际上红黑树就是一种平       
        //衡的BST
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;//一次就匹配到了，直接返回，否则进行搜索
            if ((e = first.next) != null) {
                if (first instanceof TreeNode)
                    //红黑树搜索/查找
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {
                    //链表搜索(查找)
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;//找到了就返回
                } while ((e = e.next) != null);
            }
        }
        return null;//没找到，返回null
```

### 3.put方法

1. 保证数组已经初始化，

2. 通过hashcode找到对应桶的index，查看是否有节点，没有就new
3. 如果有元素，则找到key更新value，分为红黑树和链表的查找，链表的查找到末尾没有找到就在链表的最后进行插入，插入后如果大于等于8个元素，则执行treeifyBin
4. onlyIfAbsent代表不需要更新value值，++size再判断是否需要进行resize
5. treeifyBin的链表转红黑树操作，代码中有写，注意转换条件

```java
//put函数入口,两个参数：key和value
public V put(K key, V value) {
        //下面分析这个函数，注意前3个参数，后面2个参数这里不太重要，因为所有的put后面2个参数都一样
        return putVal(hash(key), key, value, false, true);
    }
 
//下面是put函数的核心处理函数
    //hash：key的hashCode
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        //上面提到过HashMap是懒加载，所有put的时候要先检查table数组是否已经初始化了，
        //没有初始化得先初始化table数组，保证table数组一定初始化了
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;//这个函数后面有resize函数分析
   
        //到这里表示table数组一定初始化了
        //与上面get函数相同，指定key的Node，put在table数组的i=(n-1)&hash下标位置，get的时候
        //也是从table数组的该位置搜索
        if ((p = tab[i = (n - 1) & hash]) == null)
            //如果i位置还没有存储元素，则把当前的key，value封装为Node，存储在table[i]位置
            tab[i] = newNode(hash, key, value, null);
        else {
       //如果table[i]位置已经有元素了，则接下来执行的是：
       //首先判断链表或者二叉树中时候已经存在key的键值对，存在的话就更新它的value
       //不存在的话把当前的key，value插入到链表的末尾或者插入到红黑树中
       //如果链表或者红黑树中已经存在Node.key等于key，则e指向该Node，即
       //e指向一个Node：该Node的key属性与put时传入的key参数相等的那个Node，后面会更新e.value
            Node<K,V> e; K k;
 
       //为什么get和put先判断p.hash==hash,下面的if条件中去掉hash的比较也可以逻辑也正确？
       //因为hash的比较是两个整数的比较，比较的代价相对较小，key是泛型，对象的比较比整数比较
        //代价大，所以先比较hash，hash相等在比较key
            if (p.hash == hash &&//
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;//e指向一个Node：该Node的key属性与put时传入的key参数相等的那个Node
            else if (p instanceof TreeNode)
               //红黑树的插入操作，如果已经存在该key的TreeNode，则返回该TreeNode，否则返回null
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                //table[i]处存放的是链表，接下来和TreeNode类似
                //在遍历链表过程中先判断是key先前是否存在，如果存在则e指向该Node
                //否则将该Node插入到链表末尾，插入后判断链表长度是否>=8，是的话要进行额外操作
                
                //binCount最后的值是链表的长度
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                   //遍历到了链表最后一个元素,接下来执行链表的插入操作，先封装为Node再插入
                   //p指向的是链表最后一个节点，将待插入的Node置为p.next，就完成了单链表的插入
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            //TREEIFY_THRESHOLD值是8， binCount>=7,然后又插入了一个新节点，            
                            //链表长度>=8，这时要么进行扩容操作，要么把链表结构转为红黑树结构
                            //我们接下会分析treeifyBin的源码实现
                            treeifyBin(tab, hash);
                        break;
                    }
                    //当p不是指向链表末尾的时候：先判断p.key是否等于key，等于的话表示当前key
                    //已经存在了，令e指向p，停止遍历，最后会更新e的value；
                    //不等的话准备下次遍历，令p=p.next，即p=e
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                //表示当前的key之前已经存在了，并且上面的逻辑保证:e.key一定等于key
                //这是更新e.value就好
                V oldValue = e.value;//保存oldvalue
                
                //onlyIfAbsent默是false，evict为true
                //onlyIfAbsent为true表示如果之前已经存在key这个键值对了，那么后面在put这个key 
                //时，忽略这个操作，不更新先前的value，这里连接就好 
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;//更新e.value
                
                //这个函数的默认实现是“空”，即这个函数默认什么操作都不执行，那为什么要有它呢？
                //这是个hook/钩子函数，主要要在LinkedHashMap中，LinkedHashMap重写了这个函数
                //后面讲解LinkedHashMap时会详细讲解
                afterNodeAccess(e);
                return oldValue;//返回旧的value
            }
        }
        //如果是第一次插入key这个键，就会执行到这里
        ++modCount;//failFast机制
        
        //size保存的是当前HashMap中保存了多少个键值对，HashMap的size方法就是直接返回size
        //之前说过，threshold保存的是当前table数组长度*loadfactor，如果table数组中存储的
        //Node数量大于threshold，这时候会进行扩容，即将table数组的容量翻倍。后面会详细讲解
        //resize方法
        if (++size > threshold)
            resize();
        
        //这也是一个hook函数，作用和afterNodeAccess一样
        afterNodeInsertion(evict);
        return null;
    }
 
    //将链表转换为红黑树结构，在链表的插入操作后调用
/**
     * Replaces all linked nodes in bin at index for given hash unless
     * table is too small, in which case resizes instead.
     */
    final void treeifyBin(Node<K,V>[] tab, int hash) {
        int n, index; Node<K,V> e;
        
        //MIN_TREEIFY_CAPACITY值是64,也就是当链表长度>8的时候，有两种情况：
        //如果table数组的长度<64,此时进行扩容操作
        //如果table数组的长度>64，此时进行链表转红黑树结构的操作
        //具体转细节在面试中几乎没有问的，这里不细讲了，
        //大部同学认为链表长度>8一定会转换成红黑树，这是不对的！！！
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
            resize();
        else if ((e = tab[index = (n - 1) & hash]) != null) {
            TreeNode<K,V> hd = null, tl = null;
            do {
                TreeNode<K,V> p = replacementTreeNode(e, null);
                if (tl == null)
                    hd = p;
                else {
                    p.prev = tl;
                    tl.next = p;
                }
                tl = p;
            } while ((e = e.next) != null);
            if ((tab[index] = hd) != null)
                hd.treeify(tab);
        }
    }
```

### hashMap的扩容resize;

>   resize的整个过程：
>
>   1.  对oldCap，oldThr进行判断，oldCap=0代表需要初始化数组，oldThr=0就需要进行新的阈值计算
>
>   2.  然后对整个数组进行扩容，迁移
>   3.  首先判断桶位置是否有元素，如果没有，则null，如果只有一个元素则直接放到新数组e.hash & (newCap - 1)的位置，如果有多个链表元素，则进行拆分，为两个链表一个放到新数组i的位置，另外放在i+n的位置，如果是红黑树，则进入红黑树拆分的逻辑。

分开为两队链表，使用尾插法

https://www.bilibili.com/video/BV1xv411L7vr/?spm_id_from=333.788.recommend_more_video.2 的25分钟处

```java
/**
      有两种情况会调用当前函数： 
     1.之前说过HashMap是懒加载，第一次hHashMap的put方法的时候table还没初始化，
      这个时候会执行resize，进行table数组的初始化，table数组的初始容量保存在threshold中（如果从构造器中传入的一个初始容量的话），如果创建HashMap的时候没有指定容量，那么table数组的初始容量是默认值：16。即，初始化table数组的时候会执行resize函数
     2.扩容的时候会执行resize函数，当size的值>threshold的时候会触发扩容，即执行resize方法，
     这时table数组的大小会翻倍。
     注意我们每次扩容之后容量都是翻倍（*2），所以HashMap的容量一定是2的整数次幂，那么HashMap的
     容量为什么一定得是2的整数次幂呢？（面试重点） 要知道原因，首先回顾我们put
      key的时候，每一个key会对应到一个桶里面，桶的索引是这样计算的： index = hash & (n-1)，
     index的计算最为直观的想法是：hash%n，即通过取余的方式把当前的key、value键值对散列到各个桶中；
     那么这里为什么不用取余(%)的方式呢？原因是CPU对位运算支持较好，即位运算速度很快。
     
     另外,当n是2的整数次幂时：hash&(n-1)与hash%(n-1)是等价的，但是两者效率来讲是不同的，位运算的效率
     * 远高于%运算。基于这一点，HashMap中使用的是hash&(n-1)。这还带来了一个好处，就是将旧数组中的Node迁移到扩容后
     * 的新数组中的时候有一个很方便的特性：HashMap使用table数组保存Node节点，所以table数组扩容的时候（数组扩容）一定得是先重新开辟一个数组，然后把就数组中的元素重新散列到新数组中去。这里举一个例子来来说明这个特性：下面以Hash初始容量
     * n=16，默认loadfactor=0.75举例（其他2的整数次幂的容量也是类似的）：默认容量：n=16，二进制：10000；n-1：15,
     * n-1二进制：01111，即一连串1。某个时刻，map中元素大于16*0.75=12，即size>12，此时我们新建了一个数组，
     * 容量为扩容前的两倍，newtab，len=32;接下来我们需要把table中的Node搬移(rehash)到newtab。从table的i=0
     * 位置开始处理,假设我们当前要处理table数组i索引位置的node，那这个node应该放在newtab的那个位置呢？下面的hash表示node.key对应的hash值，也就等于node.hash属性值,另外为了简单，下面的hash只写出了8位（省略的高位的0），实际上
      hash是32位：node在newtab中的索引：index=hash%len=hash&(len-1)=hash&(32-1)=hash&31
      =hash&(0x0001_1111)；再看node在table数组中的索引计算：i=hash&(16-1)=hash&15
      =hash&(0x0000_1111)。注意观察两者的异同：i=hash&(0x0000_1111);index=hash&(0x0001_1111)
      这个表达式有个特点：index=hash&(0x0001_1111)=hash&(0x0000_1111) |
      hash&(0x0001_0000) =hash&(0x0000_1111) | hash&n)=i+( hash&n)。什么意思呢：
      hash&n要么等于n要么等于0;也就是：index要么等于i，要么等于i+n;再具体一点：当hash&n==0的时候，index=i;
      当hash&n==n的时候，index=i+n;这有什么用呢？当我们把table[i]位置的所有Node迁移到newtab中去的时候：
      这里面的node要么在newtab的i位置（不变），要么在newtab的i+n位置；也就是我们可以这样处理：把table[i]这个桶中的
     * node拆分为两个链表l1和类：如果hash&n==0，那么当前这个node被连接到l1链表；否则连接到l2链表。这样下来，
     * 当遍历完table[i]处的所有node的时候，我们得到两个链表l1和l2，这时我们令newtab[i]=l1,newtab[i+n]=l2,
     * 这就完成了table[i]位置所有node的迁移/rehash，这也是HashMap中容量一定的是2的整数次幂带来的方便之处。
     * 下面的resize的逻辑就是上面讲的那样。将table[i]处的Node拆分为两个链表，这两个链表再放到newtab[i]
     * 和newtab[i+n]位置
     */
    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;//保留扩容前数组引用
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {//大于0代表数组有值，
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            //正常扩容：newCap = oldCap << 1
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                //容量翻倍，扩容后的threshold自然也是*2
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        else { // =0进行数组初始化
            //table数组初始化的时候会进入到这里
            newCap = DEFAULT_INITIAL_CAPACITY;//默认容量
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);//threshold
        }
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;//更新threshold
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];//第一次扩容后的新数组
        table = newTab;//执行容量翻倍的新数组
        if (oldTab != null) {//oldTab是原始的table数组
            for (int j = 0; j < oldCap; ++j) {//之后完成oldTab中Node迁移到table中去
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null)//j这个桶位置只有一个元素，直接rehash到table数组
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                  //如果是红黑树：也是将红黑树拆分为两个链表，这里主要看链表的拆分，两者逻辑一样
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        //链表的拆分
                        Node<K,V> loHead = null, loTail = null;//第一个链表l1
                        Node<K,V> hiHead = null, hiTail = null;//第二个链表l2
                        Node<K,V> next;
                        do {
                            next = e.next;
                            if ((e.hash & oldCap) == 0) {//rehash到table[j]位置
                            //将当前node连接到l1上
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                 //将当前node连接到l2上
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
    
                        if (loTail != null) {
                            //l1放到table[j]位置
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            //l1放到table[j+oldCap]位置
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
```

### HashMap死循环

主要针对JDK1.7

https://www.bilibili.com/video/BV1z54y1i73r?from=search&seid=6381894644036141908

