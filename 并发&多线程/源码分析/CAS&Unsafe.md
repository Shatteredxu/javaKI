### Unsafe类

美团技术文章：https://tech.meituan.com/2019/02/14/talk-about-java-magic-class-unsafe.html

#### Unsafe是什么？

Unsafe是位于sun.misc包下的一个类，主要提供一些用于**执行低级别、不安全操作的方法**，如直接访问系统内存资源、自主管理内存资源等，这些方法在提升Java运行效率、增强Java语言底层资源操作能力方面起到了很大的作用。但由于Unsafe类使Java语言拥有了类似C语言指针一样操作内存空间的能力，这无疑也增加了程序发生相关指针问题的风险。在程序中过度、不正确使用Unsafe类会使得程序出错的概率变大，使得Java这种安全的语言变得不再“安全”，因此对Unsafe的使用一定要慎重。

由于Unsafe.getUnsafe会判断调用类的类加载器是否为引导类加载器，我们有两种方法可以获取Unsafe实例：一种是通过将**使用Unsafe的类交给bootstrap class loader去加载**，另一种方式是通过**反射**;

```java
//通过bootstrap class loader去加载Unsafe。
public class GetUnsafeFromMethod {
    public static void main(String[] args){
        //调用这个方法，必须要在启动类加载器中获取，否则会抛出安全异常
        Unsafe unsafe = Unsafe.getUnsafe();
        System.out.printf("addressSize=%d, pageSize=%d\n", unsafe.addressSize(), unsafe.pageSize());
    }
}
//上面代码直接使用会报安全异常SecurityException，原因是当前caller的类加载器是应用类加载器（Application Class loader），而要求的是启动类加载器，
```

或者：

```java
//通过反射获取Unsafe
import sun.misc.Unsafe;
import java.lang.reflect.Field;
public class GetUnsafeFromReflect {
    public static void main(String[] args){
        Unsafe unsafe = getUnsafe();
        System.out.printf("addressSize=%d, pageSize=%d\n", unsafe.addressSize(), unsafe.pageSize());
    }
    public static Unsafe getUnsafe() {
        Unsafe unsafe = null;
        try {
            Field f = Unsafe.class.getDeclaredField("theUnsafe");
            f.setAccessible(true);
            unsafe = (Unsafe) f.get(null);
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
        return unsafe;
    }
}
```

#### Unsafe的API

Unsafe提供的API大致可分为**内存操作**、**CAS**、Class相关、对象操作、**线程调度**、系统信息获取、内存屏障、数组操作等几类内存操作

##### 内存操作

#### CAS

比较并交换(compare and swap,CAS)，是原子操作的一种，可用于在多线程编程中实现不被打断的数据交换操作，从而避免多线程同时改写某一数据时由于执行顺序不确定性以及中断的不可预知性产生的数据不一致问题。该操作通过将内存中的值与指定数据进行比较，当数值一样时将内存中的数据替换为新的值

CAS三个问题：**1.ABA问题; 2.自旋锁开销及jdk8解决方案; 3.单对象操作及解决。**

**对于ABA问题：**使用单调递增的版本号解决，在JDK的java.util.concurrent.atomic包中提供了<u>AtomicStampedReference</u>来解决ABA问题

**自旋开销问题：****破坏掉for死循环，当超过一定时间或者一定次数时，return退出**。JDK8新增的<u>LongAddr</u>,和ConcurrentHashMap类似的方法。当多个线程竞争时，将粒度变小，将一个变量拆分为多个变量，达到多个线程访问多个资源的效果，最后再调用sum把它合起来。

**CAS只能单变量**：CAS的原子操作只能针对一个共享变量，可以用<u>AtomicReference</u>封装为对象，多个变量可以放一个自定义对象里，然后他会检查这个对象的引用是不是同一个。如果多个线程同时对一个对象变量的引用进行赋值，用AtomicReference的CAS操作可以解决并发冲突问题。