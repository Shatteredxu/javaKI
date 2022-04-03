TreeMap
-------

##### 1.属性

Entry：

```java
1.实现map接口中的Entry，
static final class Entry<K,V> implements Map.Entry<K,V> {
        K key;
        V value;
        Entry<K,V> left;
        Entry<K,V> right;
        Entry<K,V> parent;
        boolean color = BLACK;
        /**
         * Make a new cell with given key, value, and parent, and with
         * {@code null} child links, and BLACK color.
         */
        Entry(K key, V value, Entry<K,V> parent) {
            this.key = key;
            this.value = value;
            this.parent = parent;
 }
```

putAll方法：将map中的元素放入treemap

```java
public void putAll(Map<? extends K, ? extends V> map) {
        int mapSize = map.size();
  			//做一些基本的判断，得到Comparator比较器，调用父类的的putAll方法
        if (size==0 && mapSize!=0 && map instanceof SortedMap) {
            Comparator<?> c = ((SortedMap<?,?>)map).comparator();
            if (c == comparator || (c != null && c.equals(comparator))) {
                ++modCount;
                try {
                    buildFromSorted(mapSize, map.entrySet().iterator(),
                                    null, null);
                } catch (java.io.IOException cannotHappen) {
                } catch (ClassNotFoundException cannotHappen) {
                }
                return;
            }
        }
        super.putAll(map);
    }
```

##### 插入元素的Put方法

<img src="assets/16cb8859685757fdtplv-t2oaga2asx-watermark.awebp" alt="img" style="zoom:70%;" />

```java
public V put(K key, V value) {
    Entry<K,V> t = root;
    if (t == null) {//如果root为null 说明是添加第一个元素 直接实例化一个Entry 赋值给root
        compare(key, key); // type (and possibly null) check
        root = new Entry<>(key, value, null);
        size = 1;
        modCount++;
        return null;
    }
    int cmp;
    Entry<K,V> parent;//如果root不为null，说明已存在元素 
    // split comparator and comparable paths
    Comparator<? super K> cpr = comparator; 
    if (cpr != null) { //如果比较器不为null 则使用比较器
        //找到元素的插入位置
        do {
            parent = t; //parent赋值
            cmp = cpr.compare(key, t.key);
            //当前key小于节点key 向左子树查找
            if (cmp < 0)
                t = t.left;
            else if (cmp > 0)//当前key大于节点key 向右子树查找
                t = t.right;
            else //相等的情况下 直接更新节点值
                return t.setValue(value);
        } while (t != null);
    }
    else { //如果比较器为null 则使用默认比较器
        if (key == null)//如果key为null  则抛出异常
            throw new NullPointerException();
        @SuppressWarnings("unchecked")
            Comparable<? super K> k = (Comparable<? super K>) key;
       
        //找到元素的插入位置
        do {
            parent = t;
            cmp = k.compareTo(t.key);
            if (cmp < 0)
                t = t.left;
            else if (cmp > 0)
                t = t.right;
            else
                return t.setValue(value);
        } while (t != null);
    }
    Entry<K,V> e = new Entry<>(key, value, parent);//定义一个新的节点
    //根据比较结果决定插入到左子树还是右子树
    if (cmp < 0)
        parent.left = e;
    else
        parent.right = e;
    fixAfterInsertion(e);//保持红黑树性质  插入后进行修正
    size++;//元素树自增
    modCount++; 
    return null;
}
```

```java
private void fixAfterInsertion(Entry<K,V> x) {
    // 将新插入节点的颜色设置为红色
    x. color = RED;
    // while循环，保证新插入节点x不是根节点或者新插入节点x的父节点不是红色（这两种情况不需要调整）
    while (x != null && x != root && x. parent.color == RED) {
        // 如果新插入节点x的父节点是祖父节点的左孩子
        if (parentOf(x) == leftOf(parentOf (parentOf(x)))) {
            // 取得新插入节点x的叔叔节点
            Entry<K,V> y = rightOf(parentOf (parentOf(x)));
            // 如果新插入x的父节点是红色
            if (colorOf(y) == RED) {
                // 将x的父节点设置为黑色
                setColor(parentOf (x), BLACK);
                // 将x的叔叔节点设置为黑色
                setColor(y, BLACK);
                // 将x的祖父节点设置为红色
                setColor(parentOf (parentOf(x)), RED);
                // 将x指向祖父节点，如果x的祖父节点的父节点是红色，按照上面的步奏继续循环
                x = parentOf(parentOf (x));
            } else {
                // 如果新插入x的叔叔节点是黑色或缺少，且x的父节点是祖父节点的右孩子
                if (x == rightOf( parentOf(x))) {
                    // 左旋父节点
                    x = parentOf(x);
                    rotateLeft(x);
                }
                // 如果新插入x的叔叔节点是黑色或缺少，且x的父节点是祖父节点的左孩子
                // 将x的父节点设置为黑色
                setColor(parentOf (x), BLACK);
                // 将x的祖父节点设置为红色
                setColor(parentOf (parentOf(x)), RED);
                // 右旋x的祖父节点
                rotateRight( parentOf(parentOf (x)));
            }
        } else { // 如果新插入节点x的父节点是祖父节点的右孩子和上面的相似
            Entry<K,V> y = leftOf(parentOf (parentOf(x)));
            if (colorOf(y) == RED) {
                setColor(parentOf (x), BLACK);
                setColor(y, BLACK);
                setColor(parentOf (parentOf(x)), RED);
                x = parentOf(parentOf (x));
            } else {
                if (x == leftOf( parentOf(x))) {
                    x = parentOf(x);
                    rotateRight(x);
                }
                setColor(parentOf (x), BLACK);
                setColor(parentOf (parentOf(x)), RED);
                rotateLeft( parentOf(parentOf (x)));
            }
        }
    }
    // 最后将根节点设置为黑色
    root.color = BLACK;
}
```

##### 删除元素

```java
private void deleteEntry(Entry<K,V> p) {
        modCount++;
        //元素个数减一
        size--;
        // 如果被删除的节点p的左孩子和右孩子都不为空，则查找其替代节
        if (p.left != null && p. right != null) {
            // 查找p的替代节点
            Entry<K,V> s = successor (p);
            p. key = s.key ;
            p. value = s.value ;
            p = s;
        }
        Entry<K,V> replacement = (p. left != null ? p.left : p. right);
        if (replacement != null) { 
            // 将p的父节点拷贝给替代节点
            replacement. parent = p.parent ;
            // 如果替代节点p的父节点为空，也就是p为跟节点，则将replacement设置为根节点
            if (p.parent == null)
                root = replacement;
            // 如果替代节点p是其父节点的左孩子，则将replacement设置为其父节点的左孩子
            else if (p == p.parent. left)
                p. parent.left   = replacement;
            // 如果替代节点p是其父节点的左孩子，则将replacement设置为其父节点的右孩子
            else
                p. parent.right = replacement;
            // 将替代节点p的left、right、parent的指针都指向空
            p. left = p.right = p.parent = null;
            // 如果替代节点p的颜色是黑色，则需要调整红黑树以保持其平衡
            if (p.color == BLACK)
                fixAfterDeletion(replacement);
        } else if (p.parent == null) { // return if we are the only node.
            // 如果要替代节点p没有父节点，代表p为根节点，直接删除即可
            root = null;
        } else {
            // 如果p的颜色是黑色，则调整红黑树
            if (p.color == BLACK)
                fixAfterDeletion(p);
            // 下面删除替代节点p
            if (p.parent != null) {
                // 解除p的父节点对p的引用
                if (p == p.parent .left)
                    p. parent.left = null;
                else if (p == p.parent. right)
                    p. parent.right = null;
                // 解除p对p父节点的引用
                p. parent = null;
            }
        }
    }
```

##### 总结

总的来说，Treemap就是一个基于红黑树的有序map集合，可以传入自己的比较器来限定比较规则，具体其中的源码，核心主要就是实现红黑树的插入和删除操作，注意Treemap不是一个线程安全的类，并且不允许**插入为Null的key，HashMap可以为空**。