# CopyOnWriteArrayList源码学习

CopyOnWriteArrayList 是 juc 包下一个线程安全的并发容器，**底层使用数组实现**。CopyOnWrite 顾名思义是**写时复制**的意思，其基本思路是，从一开始大家都在共享同一个内容，当某个人想要修改这个内容的时候，会把内容 Copy 出去形成一个新的内容然后再进行修改，这是一种**延时懒惰策略**。

假设往一个容器添加元素的时候，不直接往当前容器添加，而是加锁后先将当前容器进行 Copy，复制出一个新的容器，然后在新的容器里添加元素，添加完元素之后，再将原容器的引用指向新的容器，最后释放锁。这样做的好处是我们可以对 CopyOnWrite 容器进行并发的读，而不需要加锁，只有在修改时才加锁，从而达到读写分离的效果。
每次对数组中的元素进行修改时都会创建一个新数组，因此**没有扩容机制**
**读元素不加锁，修改元素时才加锁**
**允许存储 null 元素**
CopyOnWriteArraySet 底层原理使用的是 CopyOnWriteArrayList
**CopyOnWriteArrayList 适用与读多写少的场景**，如果写场景比较多的场景下比较消耗内存

##### add源码

```java
public void add(int index, E element) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        //获取array
        Object[] elements = getArray();
        int len = elements.length;
        if (index > len || index < 0)
            throw new IndexOutOfBoundsException("Index: "+index+
                                                ", Size: "+len);
        Object[] newElements;
        int numMoved = len - index;
        if (numMoved == 0)//如果是添加到数组结尾
            newElements = Arrays.copyOf(elements, len + 1);//直接开辟一个len+1数组
        else {
            newElements = new Object[len + 1];
            //将index位置留出来，
            System.arraycopy(elements, 0, newElements, 0, index);
            System.arraycopy(elements, index, newElements, index + 1,
                             numMoved);
        }
        newElements[index] = element;
        setArray(newElements);//将引用指向新数组
    } finally {
        lock.unlock();
    }
}
```

##### remove源码

```java
public E remove(int index) {
        final ReentrantLock lock = this.lock;
        // 加锁
        lock.lock();
        try {
            Object[] elements = getArray();
            int len = elements.length;
            // 获取指定位置上的元素值
            E oldValue = get(elements, index);
            int numMoved = len - index - 1;
            // 如果移除的是数组中最后一个元素，复制元素时直接舍弃最后一个
            if (numMoved == 0)
                setArray(Arrays.copyOf(elements, len - 1));//复制元素时直接舍弃最后一个
            else {
                // 初始化新数组为原数组长度减 1
                Object[] newElements = new Object[len - 1];
                // 分两次完成元素拷贝
                System.arraycopy(elements, 0, newElements, 0, index);
                System.arraycopy(elements, index + 1, newElements, index,
                                 numMoved);
                // 重置 array
                setArray(newElements);
            }
            // 返回老 value
            return oldValue;
        } finally {
            // 释放锁
            lock.unlock();
        }
    }
```

##### 弱一致性迭代器

当调用iterator（）方法获取迭代器时实际上会返回一个**COWIterator对象**，COWIterator对象的snapshot变量保存了当前list的内容，cursor是遍历list时数据的下标。为什么说snapshot是list的快照呢？明明是指针传递的引用啊，而不是副本。如果在该线程使用返回的迭代器遍历元素的过程中，其他线程没有对list进行增删改，那么snapshot本身就是list的array，因为它们是引用关系。但是如果在遍历期间其他线程对该list进行了增删改，那么snapshot就是快照了，**因为增删改后list里面的数组被新数组替换了，这时候老数组被snapshot引用。这也说明获取迭代器后，使用该迭代器元素时，其他线程对该list进行的增删改不可见，因为它们操作的是两个不同的数组，这就是弱一致性**。

下面就是**COWIterator迭代器，里面保存了snapshot数组和cursor和游标，迭代器主要根据游标去访问数组

```java
static final class COWIterator<E> implements ListIterator<E> {
    /** Snapshot of the array */
    private final Object[] snapshot;
    /** Index of element to be returned by subsequent call to next.  */
    private int cursor;
    private COWIterator(Object[] elements, int initialCursor) {
        cursor = initialCursor;
        snapshot = elements;
    }
    public boolean hasNext() {
        return cursor < snapshot.length;
    }
    public boolean hasPrevious() {
        return cursor > 0;
    }
    @SuppressWarnings("unchecked")
    public E next() {
        if (! hasNext())
            throw new NoSuchElementException();
        return (E) snapshot[cursor++];
    }
    @SuppressWarnings("unchecked")
    public E previous() {
        if (! hasPrevious())
            throw new NoSuchElementException();
        return (E) snapshot[--cursor];
    }
    public int nextIndex() {return cursor;}
    public int previousIndex() {return cursor-1;}
    /**
     * Not supported. Always throws UnsupportedOperationException.
     * @throws UnsupportedOperationException always; {@code remove}
     *         is not supported by this iterator.
     */
    public void remove() {
        throw new UnsupportedOperationException();
    }
    /**
     * Not supported. Always throws UnsupportedOperationException.
     * @throws UnsupportedOperationException always; {@code set}
     *         is not supported by this iterator.
     */
    public void set(E e) {
        throw new UnsupportedOperationException();
    }
    /**
     * Not supported. Always throws UnsupportedOperationException.
     * @throws UnsupportedOperationException always; {@code add}
     *         is not supported by this iterator.
     */
    public void add(E e) {
        throw new UnsupportedOperationException();
    }
    @Override
    public void forEachRemaining(Consumer<? super E> action) {
        Objects.requireNonNull(action);
        Object[] elements = snapshot;
        final int size = elements.length;
        for (int i = cursor; i < size; i++) {
            @SuppressWarnings("unchecked") E e = (E) elements[i];
            action.accept(e);
        }
        cursor = size;
    }
}
```