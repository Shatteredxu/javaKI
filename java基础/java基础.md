java基础知识
------------

[TOC]

### 1. JDK 和 JRE 有什么区别？

JDK：java development kit java开发工具包，JDK是整个Java的核心，包括了Java运行环境(JRE)，一堆Java工具tools.jar和Java标准类库 (rt.jar)。

JRE:java runtime environment （java运行时环境），等于JVM虚拟机实现及 Java核心类库编译出来的jar包。不包含开发工具(编译器、调试器等)。

### 2. == 和 equals 的区别是什么？

***==*：** 基本类型：比较的是值是否相同；引用类型：比较的是引用是否相同。

**equals(): **对于object类就是用==比较，但是Integer和String对equals进行了重写，会去比较值是否相等。

**总结：**== 对于基本类型来说是值比较，对于引用类型来说是比较的是引用；而 equals 默认情况下是引用比较，只是很多类重新了 equals 方法，比如 String、Integer，Double，Date类等把它变成了值比较，所以基本类型比较值是否相等使用==，引用类型使用重写的equals 。

### **3. 两个对象的 hashCode相同，则 equals也一定为 true**

两个对象equals相等，则它们的hashcode必须相等。(equals相等，hashcode必相等)
		两个对象==相等，则其hashcode一定相等。

hashCode()相等即两个键值对的哈希值相等，然而哈希值相等，并不一定能得出键值对相等。

hashcode相等，但是对象可能不等，所以还需要用equals判断。

对象加入HashSet时，HashSet会先计算对象的hashcode值来判断对象加入的位置,看该位置是否有值，如果没有,HashSet会假设对象没有重复出现; 但是如果发现有值，这时会调用equals ()方法来检查两个对象是否真的相同。如果两者相同，HashSet就不会让其加入操作成功。如果不同的话，就会重新散列到其他位置。这样就大大**减少equals次数，提高执行速度**。

**重写equals，必须要重写hashcode为什么？**因为equals是用来判断对象是否相等的，而hashmap中是通过hashcode来确定数组存放的位置，按理来说equals相等的元素，存放的位置应该相等，但是如果不重写hashcode，可能会导致equals和hashcode逻辑上不一致，就会出现equals相等，但是两个元素放在不同的位置的情况

### **4. final ，static在 java 中有什么作用？**

final可以修饰引用，方法，类

**修饰引用：**

1. 如果引用为基本数据类型，则该引用为常量，该值无法修改；

2. 如果引用为引用数据类型，比如对象、数组，则该对象、数组本身可以修改，但指向对象或数组的地址引用不能修改。
3. 如果引用是类的成员变量，则必须当场赋值，否则编译会报错。

**修饰方法**不能被子类重写

**修饰类**，不能被继承，比如String。

>   #### final域重排序规则
>
>   final变量的赋值使用了`putfiled`指令，JVM会在该指令后加入<u>写屏障</u>，保证其他线程读到它的值时不会出现<u>未初始化的情况</u>
>
>   1.   final域为基本类型
>
>        **写final域基本规则：**1、JMM禁止编译器把final域的写重排序到构造函数之外；
>        2、编译器会在final域写之后，构造函数return之前，插入一个storestore屏障。这个屏障可以禁止处理器把final域的写重排序到构造函数之外。
>
>        可以确保<u>在对象引用为任意线程可见之前，对象的final域已经被正确初始化过了，而普通域不具有这个保障。</u>
>
>        **读final域重排序规则为：**在一个线程中，<u>初次读对象引用和初次读该对象包含final域</u>，JMM会禁止这两个操作的重排序，处理器会在读final域操作的前面插入一个LoadLoad屏障，实际上，读对象的引用和读该对象的final域存在间接依赖性，一般处理器不会重排序这两个操作。但是有一些处理器会重排序。因此，这条禁止重排序规则就是针对这些处理器而设定的
>
>        可以确保：在读一个对象的final域之前，<u>一定会先读这个包含这个final域的对象的引用</u>。
>
>   2.   final域为引用类型
>
>        final的成员域不能和这个final引用赋值不能重排序，也就是说要先给引用赋值，才能访问其中的成员域



static可以修饰**成员变量，成员方法，代码块**。

1. 用static关键字修饰的静态变量和不用static关键字修饰的实例变量。

2. static方法是类的方法，不需要创建对象就可以被调用，而非static方法是对象的方法，只有对象被创建出来后才可以被使用

3. static代码块在类中是独立于成员变量和成员函数的代码块的。注意： 这些static代码块只会被执行一次

### **5. String str="i"与 String str=new String(“i”)一样吗？**

细读：https://www.cnblogs.com/xiaoxi/p/6036701.html

java有关字符串的东西都在里面。

### 6. StringBuffer和Stringbuilder的区别

StringBuffer 和 StringBuilder 类的对象能够被多次的修改，并且不产生新的未使用对象。

 StringBuilder 的方法不是线程安全的（不能同步访问），StringBuffer 方法都添加了synchronized，所以是线程安全的。

所以StringBuilder 速度比 StringBuffer 快。

另外在java字符串进行拼接的时候，编译器会优化为通过**StringBuilder**来实现字符串的拼接，再toString返回。

### 7. 抽象类和接口的区别？



**抽象类：**含有abstract修饰符的class即为抽象类，abstract 类不能创建实例对象。含有abstract方法的类必须定义为abstract class，abstract class类中的方法不必是抽象的。abstract 类中定义抽象方法必须在具体(Concrete)子类中实现，所以，不能有抽象构造方法或抽象静态方法。如果子类没有实现抽象父类中的所有抽象方法，那么子类也必须定义为abstract类型。

**接口：**可以说成是抽象类的一种特例，<u>接口中的所有方法都必须是抽象的</u>。接口中的方法定义默认为public abstract类型，<u>接口中的成员变量类型默认为public static final</u>，抽象类和接口都不能直接实例化，如果要实例化，抽象类变量必须指向实现    

**两者区别：**		

1.所有抽象方法的子类对象，接口变量必须指向实现所有接口方法的类对象。

2、抽象类要被子类继承，接口要被类实现。

3、接口只能做方法申明，抽象类中可以做方法申明，也可以做方法实现

4、接口里定义的变量只能是公共的静态的常量，抽象类中的变量是普通变量。

5、抽象类里的抽象方法必须全部被子类所实现，如果子类不能全部实现父类抽象方法，那么该子类只能是抽象类。同样，一个实现接口的时候，如不能全部实现接口方法，那么该类也只能为抽象类。

6、抽象方法只能申明，不能实现，接口是设计的结果 ，抽象类是重构的结果

7、抽象类里可以没有抽象方法

8、如果一个类里有抽象方法，那么这个类只能是抽象类

9、抽象方法要被实现，所以不能是静态的，也不能是私有的。

10、接口可继承接口，并可多继承接口，但类只能单根继承

**注意**：JDK 1.8 以后，接口里可以有静态方法和方法体了。

### 8. 普通类和抽象类区别？

· 抽象类不能被实例化

· 抽象类可以有抽象方法，抽象方法只需申明，无需实现

· 含有抽象方法的类必须申明为抽象类

· 抽象的子类必须实现抽象类中所有抽象方法，否则这个子类也是抽象类

· 抽象方法不能被声明为静态

· 抽象方法不能用private修饰

· 抽象方法不能用final修饰

### 9. 容器都有哪些？

Java集合是java提供的工具包，包含了常用的数据结构：**集合、链表、队列、栈、数组、映射**等。Java集合工具包位置是java.util.*。
		Java集合主要可以划分为4个部分：List列表、Set集合、Map映射、工具类(Iterator迭代器、Enumeration枚举类、Arrays和Collections)、。
		Java集合工具包框架图(如下)：

![集合框架总图](assets/集合框架总图.jpg)

### 10.java异常处理机制

细读：https://link.juejin.cn/?target=http%3A%2F%2Fblog.csdn.net%2Fhguisu%2Farticle%2Fdetails%2F6155636

### 11. java中null

1.null是任何引用类型的默认值

2.null既不是对象也不是一种类型，它仅是一种特殊的值，你可以将其赋予任何引用类型，你也可以将null转化成任何类型

3.null可以赋值给引用变量，不能将null赋值给基本类型变量，如int、double、float、boolean。

4.任何含有null值的包装类在java拆箱生成基本数据类型时候都会抛出一个**空指针异常**。

5.**null是任何一个引用类型变量的默认值，在java中不能使用null引用来调用任何instance方法或者instance变量。**

### 12.重载和重写区别

***重写：***

1.发生在父类与子类之间
       2.方法名，参数列表，返回类型（除过子类中方法的返回类型是父类中返回类型的子类）必须相同
  	  3.访问修饰符的限制一定要大于被重写方法的访问修饰符（public>protected>default>private)
	    4.重写方法一定不能抛出新的检查异常或者比被重写方法申明更加宽泛的检查型异常

***重载：***

1.发生在同一个类中，

2.方法名必须相同，参数类型不同，个数不同，顺序不同，方法返回值和返回修饰符可以不同。

***总结：***

1、重写实现的是**运行时的多态**，编译时看不出子类调用的是哪个方法，但是运行时操作数栈会先根据子类的引用去子类的类信息中查找方法，找不到的话再到父类的类信息中查找方法。而重载实现的是**编译时的多态**，编译期就可以确定传入的参数组合，决定调用的具体方法是哪一个了。

2、重写的方法参数列表必须相同；而重载的方法参数列表必须不同。 

3、重写的方法的返回值类型只能是**父类类型或者父类类型的子类**，而重载的方法对返回值类型没有要求

### 13. java中三大特性

***继承：***将几个类的公共的属性或方法提取出来，新建一个父类放入其中，然后让那几个类都继承这个父类中的属性或方法就可以了。(注意： 一个类只能继承一个类，而不能多继承，子类只能获取父类中的非private修饰的变量或方法，子类可以重写父类中的方法，覆盖父类中的属性，如果不重写的话，就会调用父类中对应的方法或属性)

**目的**就是减少代码的冗余，代码复用。

***封装：***将一个对象的成员变量私有化，并对外只提供改变或获得该私有变量值的方法（get、set），从而控制成员变量的访问级别，**目的**就是提高代码的安全性，不会再暴露类的内部成员变量属性，封装也使使用者更加快捷方便的使用，不用在意其内部实现。

***多态：***

**编译时多态：**方法重载

**运行时多态：**JAVA运行时系统根据调用该方法的实例的类型来决定选择调用哪个方法则被称为运行时多态，(1)要有继承(包括接口的实现);(2)要有重写;(2)父类引用指向子类对象。

下面我们看一道面试题：

```java
class A { 
     public String show(D obj)...{ 
        return ("A and D"); 
     } 
     public String show(A obj)...{ 
        return ("A and A"); 
     } 
} 
class B extends A{ 
     public String show(B obj)...{ 
        return ("B and B"); 
     } 
     public String show(A obj)...{ 
        return ("B and A"); 
     } 
} 
class C extends B...{} 
class D extends B...{}
以下输出结果是什么？
        A a1 = new A(); 
        A a2 = new B(); 
        B b = new B(); 
        C c = new C(); 
        D d = new D(); 
        System.out.println(a1.show(b));   ① 
        // a1是A类的一个实例化对象，所以this指向A，然后查找this.show(b),由于没有这个方法，所以到super.show(b)，但是由于A类没有超类了，所以到this.show(super b),由于b的超类是A，所以相当于this.show(A),然后在A类中查找到了这个方法，于是输出 A and A。
        System.out.println(a1.show(c));   ② 
        // 同样，a1是A类的实例化对象，所以this指向A，然后在A类中查找this.show(C)方法，由于没有这个方法，所以到了super.show(C),由于A类的超类里面找，但是A没有超类，所以到了this.show(super C)，由于C的超类是B所以在A类里面查找this.show(B)方法，也没找到，然后B也有超类，就是A，所以查找this.show(A),找到了，于是输出 A and A// 
        System.out.println(a1.show(d));   ③ 
        //在A类中找到this.show(D)方法,所以就输出A and D;
        System.out.println(a2.show(b));   ④ 
        // a2是B类的引用对象，类型为A，所以this指向A类，然后在A类里面找this.show(B)方法，没有找到，所以到了super.show(B),由于A类没有超类，所以到了this.show(super B)，B的超类是A，即super B = A，所以执行方法this。show(A)，在A方法里面找show(A)，找到了，但是由于a2是一个类B的引用对象，而B类里面覆盖了A类的show(A)方法，所以最终执行的是B类里面的show(A)方法，即输出B and A;
        System.out.println(a2.show(c));   ⑤ 
        //　a2是B类的引用对象，类型为A，所以this指向A类，然后在A类里面找this.show(C)方法，没有找到，所以到了super.show(C)方法，由于A类没有超类，所以到了this.show(super C),C的超类是B，所以在A类里面找show(B)，同样没有找到，发现B还有超类，即A，所以还继续在A类里面找show(A)方法，找到了，但是由于a2是一个类B的引用对象，而B类里面覆盖了A类的show(A)方法，所以最终执行的是B类里面的show(A)方法，即输出B and A;
        System.out.println(a2.show(d));   ⑥ 
        //a2是B类的引用对象，类型为A，所以this指向A类，然后在A类里面找this.show(D)方法，找到了，但是由于a2是一个类B的引用对象，所以在B类里面查找有没有覆盖show(D)方法，没有，所以执行的是A类里面的show(D)方法，即输出A and D;
        System.out.println(b.show(b));    ⑦ 
        //b是B类的一个实例化对象，首先执行this.show(B)，在B类里面找show(B)方法，找到了，直接输出B and B;
        System.out.println(b.show(c));    ⑧ 
        //b是B类的一个实例化对象，首先执行this.show(C)，在B类里面找show(C)方法，没有找到，所以到了super.show(c),B的超类是A，所以在A类中找show(C)方法，没有找到，于是到了this.show(super C),C的超类是B，所以在B类中找show(B)f方法，找到了，所以执行B类中的show(B)方法输出B and B;
        System.out.println(b.show(d));    ⑨
        //b是B类的一个实例化对象，首相执行this.show(D)，在B类里面找show(D)方法，没有找到，于是到了super.show(D),B的超类是A类，所以在A类里面找show(D)方法，找到了，输出A and D;
      
 //所以总过程就是对于a.show(b)从this.show(b)开始，super a.show(b)到a.show(super b)最后super a.show(super b)
```

***编译看左，运行看右***：

以Person s=new Student();为例。左边用以**声明类型**，右边用以**创建对象**。而编译器编译时会查看左边的声明中是否有编译错误（在多态中尤为重要，看是否左边的类型中是否缺少右边类型的方法，否则报错。右边的类型通常为左边类型的子类）。即使没有报错，一旦右边实际真正运行起来也有可能会出现错误，这时就要查看右边类中具体实现的代码。所以编**译时错误看左边，运行时错误看右边。**

### 14.一个Java程序的运行过程（TODO）

***《TODO，写完JVM再来总结》***

 第一步：将java源码(.java文件)通过编译器(javac.exe)编译成JVM文件(.class文件)

使用javac命令，

 第二步：将JVM文件通过java.exe执行，输出结果

### 15.BIO、NIO、AIO 有什么区别？

同步阻塞的BIO、同步非阻塞的NIO、异步非阻塞的AIO

转–>[](../经典面试题.md)的23.BIO，NIO，AIO区别

### 16.unicode和UTF8区别

细读：https://blog.csdn.net/qq_36761831/article/details/82291166

### 17.Int和integer的区别，自动拆箱，装箱是怎么实现的。

在装箱的时候自动调用的是*Integer的valueOf(int)方法*。而在拆箱的时候自动调用的是*Integer的intValue方法*。

对于每个基本类型，都会有对应的引用类型。int->Integer,byte->Byte, short→Short,long---›Long, float→Float,double→Double，他们内部都有两个方法：

**静态方法valueOf(基本类型)**：将给定的基本类型转换成对应的包装类型；

**实例方法xxxValue()：**将具体的包装类型对象转换成基本类型； 

##### Integer源码：

```java
public final class Integer extends Number implements Comparable<Integer> {
private final int value;

/*Integer的构造方法，接受一个整型参数,Integer对象表示的int值，保存在value中*/
 public Integer(int value) {
        this.value = value;
 }
/*equals()方法判断的是:所代表的int型的值是否相等*/
 public boolean equals(Object obj) {
        if (obj instanceof Integer) {
            return value == ((Integer)obj).intValue();
        }
        return false;
}
/*返回这个Integer对象代表的int值，也就是保存在value中的值*/
 public int intValue() {
        return value;
 }
 /**
  * 首先会判断i是否在[IntegerCache.low,Integer.high]之间
  * 如果是，直接返回Integer.cache中相应的元素
  * 否则，调用构造方法，创建一个新的Integer对象
  */
 public static Integer valueOf(int i) {
    assert IntegerCache.high >= 127;
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
 }
/**
  * 静态内部类，缓存了从[low,high]对应的Integer对象
  * low -128这个值不会被改变
  * high 默认是127，可以改变，最大不超过：Integer.MAX_VALUE - (-low) -1
  * cache 保存从[low,high]对象的Integer对象
 */
 private static class IntegerCache {
    static final int low = -128;
    static final int high;
    static final Integer cache[];

    static {
        // high value may be configured by property
        int h = 127;
        String integerCacheHighPropValue =
            sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
        if (integerCacheHighPropValue != null) {
            int i = parseInt(integerCacheHighPropValue);
            i = Math.max(i, 127);
            // Maximum array size is Integer.MAX_VALUE
            h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
        }
        high = h;
 
        cache = new Integer[(high - low) + 1];
        int j = low;
        for(int k = 0; k < cache.length; k++)
            cache[k] = new Integer(j++);
    }
    private IntegerCache() {}
}
```

**<u>注意只有使用integer.valueOf的时候才回村对象  如果使用 new则会创建对象</u>**

以上是Oracle(Sun)公司JDK 1.7中Integer源码的一部分，通过分析上面的代码，得到：

1）Integer有一个实例域value，它保存了这个Integer所代表的int型的值，且它是**final的**，也就是说这个Integer对象一经构造完成，它所代表的值就不能再被改变。

2）Integer**重写了equals()方法**，它通过比较两个Integer对象的value，来判断是否相等。

3）重点是静态内部类IntegerCache，通过类名就可以发现：它是用来缓存数据的。它有一个数组，里面保存的是连续的Integer对象。 (a) low：代表缓存数据中最小的值，固定是-128。

(b) high：代表缓存数据中最大的值，它可以被该改变，默认是127。high最小是127，最大是Integer.MAX_VALUE-(-low)-1，如果high超过了这个值，那么cache[ ]的长度就超过Integer.MAX_VALUE了，也就溢出了。

(c) cache[]：里面保存着从[low,high]所对应的Integer对象，长度是high-low+1(因为有元素0，所以要加1)。

4）调用valueOf(inti)方法时，首先判断i是否在[low,high]之间，如果是，则复用Integer.cache[i-low]。比如，如果Integer.valueOf(3)，直接返回Integer.cache[131]；如果i不在这个范围，则调用构造方法，构造出一个新的Integer对象。

5）调用intValue()，直接返回value的值。 通过3）和4）可以发现，默认情况下，在使用自动装箱时，VM会复用[-128,127]之间的Integer对象。

##### 总结：

通过上面的代码，我们分析一下自动装箱与拆箱发生的时机：

（1）当需要一个对象的时候会自动装箱，比如Integer a = 10;equals(Object o)方法的参数是Object对象，所以需要装箱。

（2）当需要一个基本类型时会自动拆箱，比如int a = new Integer(10);算术运算是在基本类型间进行的，所以当遇到算术运算时会自动拆箱，比如代码中的 c == (a + b);

（3） 包装类型 **==** 基本类型时，包装类型自动拆箱；

需要注意的是：“==”在没遇到算术运算时，不会自动拆箱；基本类型只会自动装箱为对应的包装类型，代码中最后一条说明的内容。

在JDK 1.5中提供了自动装箱与自动拆箱，这其实是Java 编译器的语法糖，编译器通过调用包装类型的valueOf()方法实现自动装箱，调用xxxValue()方法自动拆箱。自动装箱和拆箱会有一些陷阱，那就是包装类型复用了某些对象。

（1）Integer默认复用了[-128,127]这些对象，其中高位置可以修改；

（2）Byte复用了全部256个对象[-128,127]；

（3）Short复用了[-128,127]这些对象；

（4）Long复用了[-128,127];

（5）Character复用了[0,127],Charater不能表示负数;

**Double和Float是连续不可数的，所以没法复用对象，也就不存在自动装箱复用陷阱。**

Boolean没有自动装箱与拆箱，它也复用了Boolean.TRUE和Boolean.FALSE，通过Boolean.valueOf(boolean b)返回的Blooean对象要么是TRUE，要么是FALSE，这点也要注意。

### 18. java 8&9&10&11新特性

###### java 8特性

https://www.jianshu.com/p/5b800057f2d8

学习：https://github.com/h2pl/Java-Tutorial/blob/master/docs/java/basic/21%E3%80%81Java8%E6%96%B0%E7%89%B9%E6%80%A7%E7%BB%88%E6%9E%81%E6%8C%87%E5%8D%97.md

1. Lambda 表达式

    创建一个线程：`new Thread(() -> System.out.println("Hello World")).start();`

    自定义排序方法：都是使用Lambda 表达式

    ```java
    					//不使用Lambda 表达式
    					Collections.sort(names, new Comparator<String>() {
                @Override
                public int compare(String a, String b) {
                    return b.compareTo(a);
                }
            });
    				//使用Lambda 表达式
    				Collections.sort(names, (String a, String b) -> {
                return b.compareTo(a);
            });
            Collections.sort(names, (String a, String b) -> b.compareTo(a));
            Collections.sort(names, (a, b) -> b.compareTo(a));
    ```

2. 接口的默认方法和静态方法

3. 方法引用

   构造器应用，静态方法引用，成员方法的引用，实例对象的成员方法的引用

4. 重复注解

    相同的注解可以在同一地方声明多次

5. 方法参数名字可以反射获取

6. Date/Time API

7. Stream API

    **函数式编程方法**

8. Optional

    处理空值

9. Nashorn JavaScript引擎

   可以在JVM上运行JavaScript应用

10. Base64

###### java 9 特性

1. Java 平台级模块系统

   通过 *requires* 来表示需要的依赖。 *exports* 语句控制着哪些包是可以被其它模块访问到的。所有不被导出的包默认都封装在模块的里面。

2. JShell交互式编程环境

3. 改进的 Stream API

### 19.深拷贝和浅拷贝

细读：https://www.cnblogs.com/ysocean/p/8482979.html

总结：主要从下面几个方面回答：

1. **浅拷贝就是**先创建一个新对象，然后将当前对象的非静态字段复制到该新对象，如果字段是值类型的，那么对该字段执行复制；如果该字段是引用类型的话，则复制引用但不复制引用的对象。因此，原始对象及其副本引用同一个对象。

2. **深拷贝就是** 创建一个新对象，然后将当前对象的非静态字段复制到该新对象，无论该字段是值类型的还是引用类型，都复制独立的一份。当你修改其中一个对象的任何内容时，都不会影响另一个对象的内容。

3. **如何更好的实现深拷贝？**1. 让每个引用类型属性内部都重写clone() 方法2. 利用对象的序列化产生克隆对象，然后通过反序列化获取这个对象（不需要序列化的属性，将其声明为 **transient**，即将其排除在克隆属性之外）3. Effective Java 书上讲到，最好不要去使用 clone()，可以使用**拷贝构造函数或者拷贝工厂来拷贝一个对象**，就像下面这个：

   ```java
   public class CloneConstructorExample {
   
       private int[] arr;
   
       public CloneConstructorExample() {
           arr = new int[10];
           for (int i = 0; i < arr.length; i++) {
               arr[i] = i;
           }
       }
       public CloneConstructorExample(CloneConstructorExample original) {//通过另外的构造方法，来实现深拷贝
           arr = new int[original.arr.length];
           for (int i = 0; i < original.arr.length; i++) {
               arr[i] = original.arr[i];//依次进行赋值
           }
       }
       public void set(int index, int value) {
           arr[index] = value;
       }
       public int get(int index) {
           return arr[index];
       }
   }
   CloneConstructorExample e1 = new CloneConstructorExample();
   CloneConstructorExample e2 = new CloneConstructorExample(e1);
   e1.set(2, 222);//e1更改了，下面的e2没改
   System.out.println(e2.get(2)); // 2
   ```

### 20.new Integer(123) 与 Integer.valueOf(123) 区别？

细读：https://blog.csdn.net/liufangbaishi2014/article/details/52792086

```java
// new Integer每次出来都是不同的对象
// 但是Integer.valueOf(i)当i >= -128 && i <= 127时，使用缓存机制，返回的是同一个对象。
public static void main(String[] args) {
        Integer a = new Integer(1);
        Integer b = new Integer(1);
        System.out.println("a==b? " + (a==b));
        Integer c = Integer.valueOf(1);
        Integer d = Integer.valueOf(1);
        System.out.println("c==d? " + (c==d));
    }
//a==b? false
//c==d? true
```



### 21.java中的类型转换

自动类型转换：

>   Byte→short→int→long---›float→double
>
>   小的类型自动转化为大的类型。
>
>   整数类型可以自动转化为浮点类型，可能会产生舍入误差。

强制类型转换：

>    大类型转小类型，在大类型前加（小类型）强制转换，可能会丢失精度（舍弃小数点等）



### 22.面向对象和面向过程区别

*   ***面向过程：***该思想是站着过程的角度思考问题，强调的就是功能行为，功能的执行过程，即先后顺序，而每一个功能我们都使用函数（类似于方法）把这些步骤一步一步实现。使用的时候依次调用函数就可以了。
*   ***面向对象：***将问题分解成各个对象，来描叙某个事物在整个解决问题的步骤中的行为。

例如五子棋，面向过程的设计思路就是首先分析问题的步骤：1、开始游戏，2、黑子先走，3、绘制画面，4、判断输赢，5、轮到白子，6、绘制画面，7、判断输赢，8、返回步骤2，9、输出最后结果。把上面每个步骤用不同的方法来实现，我们只需要关心每一个功能函数的编写。

面向对象的设计则是整个五子棋可以分为 1、**黑白双方**，这两方的行为是一模一样的，2、**棋盘系统**，负责绘制画面，3、**规则系统**，负责判定诸如犯规、输赢等。第一类对象（玩家对象）负责接受用户输入，并告知第二类对象（棋盘对象）棋子布局的变化，棋盘对象接收到了棋子的变化就要负责在屏幕上面显示出这种变化，同时利用第三类对象（规则系统）来对棋局进行判定。所以我们只需把问题抽象为，什么对象（类），有什么属性（成员变量），干了什么事情（函数方法）。

### 23.向上转型和向下转型

向上转型和向下转型都是**针对引用的转型**，是**编译期进行的转型**，根据引用类型来判断使用哪个方法。并且在传入方法时会自动进行转型（有需要的话）。运行期将引用指向实例，如果是不安全的转型则会报错，若安全则继续执行方法。也就是我们常说的：编译看左，运行看右。

**向上转型：**

```java
		假设Son是Father的子类，Son中方法有：smoke(),play();Father中方法有：smoke()，drive（）
		Son son = new Son();
    //首先先明确一点，转型指的是左侧引用的改变。
    //father引用类型是Father，指向Son实例，就是向上转型，既可以使用子类的方法，也可以使用父类的方法。
    Father father = son;//向上转型,此时运行father的方法
    father.smoke();
    //不能使用子类独有的方法。
    // father.play();编译会报错
    father.drive();
    //Son类型的引用指向Father的实例，所以是向下转型，不能使用子类非重写的方法，可以使用父类的方法。
```

向下转型：两种情况：如果是先向上转型，再向下，则没有报错；如果本身就是父类，直接转为子类，调用子类方法，则编译不错，运行错误（ClassCastException错误）。

```java
//向下转型，此时运行了son的方法
    Son son1 = (Son) father;
    //转型后就是一个正常的Son实例
    son1.play();
    son1.drive();
    son1.smoke();
    
   //因为向下转型之前必须先经历向上转型。
	//在向下转型过程中，分为两种情况：

	//情况一：如果父类引用的对象如果引用的是指向的子类对象，
	//那么在向下转型的过程中是安全的。也就是编译是不会出错误的。
    //因为运行期Son实例确实有这些方法
    Father f1 = new Son();
    Son s1 = (Son) f1;
    s1.smoke();
    s1.drive();
    s1.play();

    //情况二：如果父类引用的对象是父类本身，那么在向下转型的过程中是不安全的，编译不会出错，
    //但是运行时会出现java.lang.ClassCastException错误。它可以使用instanceof来避免出错此类错误。
    //因为运行期Father实例并没有这些方法。
        Father f2 = new Father();
        Son s2 = (Son) f2;
        s2.drive();
        s2.smoke();
        s2.play();
```

### 24.构造方法，代码块，静态代码块执行顺序

```java
class A {
    public A() {
        System.out.println("1A类的构造方法");
    }
    {
        System.out.println("2A类的构造快");
    }
    static {
        System.out.println("3A类的静态块");
    }
}
 
public class B extends A {
    public B() {
        System.out.println("4B类的构造方法");
    }
    {
        System.out.println("5B类的构造快");
    }
    static {
        System.out.println("6B类的静态块");
    }
    public static void main(String[] args) {
        System.out.println("7");
        new B();
        new B();
        System.out.println("8");
    }
}
```

<u>执行顺序结果为：367 2154 2154 8</u>

执行顺序依次为：

>   1父类的静态成员和代码块
>          2子类静态成员和代码块
>          3父类成员初始化和代码快
>    	   4父类构造方法
>   		5子类成员初始化和代码块
>   		6子类构造方法



每次new都会执行构造方法以及构造块。 构造块的内容会在构造方法之前执行。 非主类的静态块会在类加载时，构造方法和构造块之前执行，切只执行一次。 主类（public class）里的静态块会先于main执行。 继承中，子类实例化，会先执行父类的构造方法，产生父类对象，再调用子类构造方法。 所以题目里，由于主类B继承A，所以会先加载A，所以第一个执行的是第3句。

从第4点我们知道6会在7之前执行，所以前三句是367。

之后实例化了B两次，每次都会先实例化他的父类A，然后再实例化B，而根据第1、2、5点，知道顺序为2154。

最后执行8。

所以顺序是367215421548

### 24.值传递和引用传递

### 字符串Intern方法

```java
				String s1 = new StringBuilder("go").append("od").toString();
        System.out.println(s1.intern() == s1);//true
        String s2 = new StringBuilder("ja").append("va").toString();
        System.out.println(s2.intern() == s2);//false
```

JDK 1.7（以及部分其他虚拟机，例如JRockit）的intern()实现不会再复制实例，只是在常量池中记录首次使用intern出现的实例引用，因此intern()返回的引用和由StringBuilder创建的那个字符串实例是同一个。

### Java泛型

泛型的基本原理是类型擦除，即编译器会在编译期间「擦除」泛型语法并相应的做出一些类型转换动作，对于大部分的泛型类型都以 Object 进行擦除，而有一种情况则不是。那就是使用到了extends和super语法的有界类型`Caculate<T extends String>`这种情况下会替换为String类而不是object类，交给虚拟机的class文件并没有泛型

### 对象引用与对象存放的地址和区别

`DemoClass demo= new DemoClass ();`包含了四个动作。
			1）右边的“ new DemoClass”，是以DemoClass类为模板，**在堆空间里创建一个DemoClass类对象**（也简称为DemoClass对象）。
			2）末尾的()意味着，在对象创建后，**立即调用DemoClass类的构造函数，对刚生成的对象进行初始化。**
			3）左边的“DemoClass”创建了一个**DemoClass类引用变量**。所谓DemoClass类引用，就是以后可以用来指向DemoClass对象的对象引用。
			4）“=”操作符使对象引用指向刚创建的那个DemoClass对象。

对象一般存储在堆中，而引用存储在速度更快的堆栈中。

### 反射

一般我们使用new初始化一个对象，然后调用它的方法属性，这叫正射，但是如果我需要想要在运行时对访问java对象的属性，方法，构造方法，那就可以用到反射，主要用途有：

1. **开发通用框架**，比如Spring的JavaBean ，我们可能需要根据配置文件加载不同的对象或类，调用不同的方法，这个时候就必须用到反射——运行时动态加载需要加载的对象。
2. 动态代理：拦截特定方法，添加一些属性
3. 注解：
4. 扩展功能：

缺点：

- **性能开销** - 由于反射涉及动态解析的类型，因此无法执行某些 Java 虚拟机优化。因此，反射操作的性能要比非反射操作的性能要差，应该在性能敏感的应用程序中频繁调用的代码段中避免。
- **破坏封装性** - 反射调用方法时可以忽略权限检查，因此可能会破坏封装性而导致安全问题。
- **内部曝光** - 由于反射允许代码执行在非反射代码中非法的操作，例如访问私有字段和方法，所以反射的使用可能会导致意想不到的副作用，这可能会导致代码功能失常并可能破坏可移植性。反射代码打破了抽象，因此可能会随着平台的升级而改变行为。

### 序列化原理

**Java序列化**是指把Java对象转换为字节序列的过程；而**Java反序列化**是指把字节序列恢复为Java对象的过程。网络通信的时候，发送各种类型的数据都是通过二进制流序列的形式在网络上传送，然后通过反序列化解析为原对象，通过序列化之后，我们也可以将实现数据的持久化。

### Java常量池

Java中的常量池，分为两种：静态常量池和运行时常量池。

- 静态常量池 ： *.class文件中的常量池，class文件中的常量池不仅仅包含字符串(数字)字面量，还包含类、方法的信息，占用class文件绝大部分空间。（编译时期）
- 运行时常量池 ： jvm虚拟机在完成类装载操作后，将class文件中的常量池载入到内存中，并保存在方法区中，我们常说的常量池，就是指方法区中的运行时常量池。（运行时期）
  补充 ： 运行时常量池中的常量，基本来源于各个class文件中的常量池。（即每个class文件都有对应的常量池）

对于大部分包装类常量池，都实现了常量池技术，将一些对象缓存在常量池中

final修饰的成员变量（实例变量）和静态变量（静态变量也只能是用static修饰的成员变量）存储在类的常量池中，那么用final修饰的局部变量（方法内）我们也可以称之为不可变变量。（存储在栈中）

### 怎样计算一个 Java 对象大小？

一个 Java 对象内存占用由三部分组成：`对象头（Header）`,`实例数据（Instance Data）`和`对齐填充（Padding）`

##### 对象头：

存储`对象自身的运行时数据`，如 `hashCode` 、`GC分代年龄`、`锁状态标志`、`线程持有的锁`、`偏向线程ID`、`偏向时间戳`，这部分数据的长度在 32 位和 64 位的虚拟机（未开启指针压缩）中分别为 4B 和 8B ，官方称之为 `”Mark Word”`

对象的另一部分是`类型指针（kclass）`，即对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是那个类的实例。另外如果对象是一个 Java 数组，那在对象头中还必须有一块用于记录数组长度的数据，因为虚拟机可以通过普通 Java 对象的元数据信息确定 Java 对象的大小，但是从数组的元数据中却无法确定数组的大小。同样，<u>这部分数据的长度在 32 位和 64 的虚拟机（未开启指针压缩）中分别为 4B 和 8B</u>。

> 32位虚拟机对象头大小= Mark Word（4B）+ kclass(4B) = 8B   
>
>  64位虚拟机对象头大小= Mark Word（8B）+ kclass(4B) = 12B

##### 实例数据

根据基本类型（不同基本类型不一样）和引用类型（4个字节）

##### 对象填充

Java 对象大小必须为 8 的整数倍，当对象头+实例数据大小不为 8 的整数倍时，将会使用Padding机制进行填充

##### 数组的大小

Java 中数组也是一种对象，数组的大小与普通 Java 对象相比多了数组长度的信息（4B），即一个数组对象大小为 Mark Word（8B）+ kclass(4B)[开启指针压缩] + 数组长度(4B) = 16B