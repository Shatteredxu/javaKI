JVM类加载机制
-------------

**类加载过程：**加载验证准备解析初始化

**类加载顺序：**先静态代码块，再其他

类加载器：Bootstrap ClassLoader，Extention ClassLoader，App ClassLoader

双亲委派：

##### 打破双亲委派机制

 WebAppClassLoader，它加载自己目录下的 .class 文件，并不会传递给父类的加载器。但是，它却可以使用 SharedClassLoader 所加载的类，实现了共享和分离的功能。

<img src="assets/Cgq2xl4cQNeAZ4FuAABzsqSozok762.png" alt="img" style="zoom:80%;" />

Java 中有一个 **SPI 机制**，全称是 Service Provider Interface，是 Java 提供的一套用来被第三方实现或者扩展的 API，它可以用来启用框架扩展和替换组件。

**OSGi**

##### 替换 JDK 的类

要使用到 Java 的 endorsed 技术。我们需要将自己的 HashMap 类，打包成一个 jar 包，然后放到 -Djava.endorsed.dirs 指定的目录中。注意类名和包名，应该和 JDK 自带的是一样的。但是，java.lang 包下面的类除外，因为这些都是特殊保护的。

