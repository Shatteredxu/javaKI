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

### **4. final ，static在 java 中有什么作用？**

final可以修饰引用，方法，类

**修饰引用：**

1. 如果引用为基本数据类型，则该引用为常量，该值无法修改；

2. 如果引用为引用数据类型，比如对象、数组，则该对象、数组本身可以修改，但指向对象或数组的地址引用不能修改。
3. 如果引用时类的成员变量，则必须当场赋值，否则编译会报错。

**修饰方法**不能被子类重写

**修饰类**，不能被继承，比如String。



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

### 7. 抽象类和接口的区别

含有abstract修饰符的class即为抽象类，abstract 类不能创建实例对象。含有abstract方法的类必须定义为abstract class，abstract class类中的方法不必是抽象的。abstract 类中定义抽象方法必须在具体(Concrete)子类中实现，所以，不能有抽象构造方法或抽象静态方法。如果子类没有实现抽象父类中的所有抽象方法，那么子类也必须定义为abstract类型。

接口（interface）可以说成是抽象类的一种特例，<u>接口中的所有方法都必须是抽象的</u>。接口中的方法定义默认为public abstract类型，<u>接口中的成员变量类型默认为public static final</u>，抽象类和接口都不能直接实例化，如果要实例化，抽象类变量必须指向实现    		

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



### 8. 普通类和抽象类有哪些区别？

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

### 11. java中null；

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

1、重写实现的是**运行时的多态**，而重载实现的是**编译时的多态**。 

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
        //b是B类的一个实例化对象，首相执行this.show(B)，在B类里面找show(B)方法，找到了，直接输出B and B;
        System.out.println(b.show(c));    ⑧ 
        //b是B类的一个实例化对象，首相执行this.show(C)，在B类里面找show(C)方法，没有找到，所以到了super.show(c),B的超类是A，所以在A类中找show(C)方法，没有找到，于是到了this.show(super C),C的超类是B，所以在B类中找show(B)f方法，找到了，所以执行B类中的show(B)方法输出B and B;
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

转->[](../经典面试题.md)的23.BIO，NIO，AIO区别

### 16.unicode和UTF8区别

细读：https://blog.csdn.net/qq_36761831/article/details/82291166

### 17.Int和integer的区别. 自动拆箱，装箱是怎么实现的

a 和 b的取值只要在-128到127之间，那么他们指的就是同一个，即使==比较的是两者的引用，两者也是相同的，因为-128到127这些数字是使用频率比较高的，就产生了一个整型常量池，这些数字会存放在这里，有相同的数字则不会再次创建，所以a和b指的是同一个，因此两者相同，当然如果是在这个范围之外的数字，那结果就是false了

在装箱的时候自动调用的是*Integer的valueOf(int)方法*。而在拆箱的时候自动调用的是*Integer的intValue方法*。

### 18. java 8&9&10&11新特性

###### java 8特性

https://www.jianshu.com/p/5b800057f2d8

1. Lambda 表达式

2. 接口的默认方法和静态方法

3. 方法引用

   构造器应用，静态方法引用，成员方法的引用，实例对象的成员方法的引用

4. 重复注解

5. Date/Time API

6. Stream API

7. Optional

8. Nashorn JavaScript引擎

   可以在JVM上运行JavaScript应用

9. Base64

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

### 21.java中的类型转换

