# JVM参数

##  1.堆内存相关

> Java 虚拟机所管理的内存中最大的一块，Java 堆是所有线程共享的一块内存区域，在虚拟机启动时创建。**此内存区域的唯一目的就是存放对象实例，几乎所有的对象实例以及数组都在这里分配内存。**

### 2.1 显式指定堆内存`–Xms`和`-Xmx`

```text
-Xms<heap size>[unit] 指定最小堆内存
-Xmx<大小>[单位] 指定最大堆内存
-XX：+HeapDumpOnOutOf-MemoryError可以让虚拟机在出现内存溢出异常的时候Dump出当前的内存堆转储快照以便进行事后分析
```

### 2.2设置新生代内存

> ```text
> -XX:NewSize=256m最小新生代内存
> -XX:MaxNewSize=1024m最大新生代内存
> -XX:NewRatio=1控制新生代和老年代的比例
> ```

### 2.3显式指定永久代/元空间的大小

> -XX:PermSize=N //方法区 (永久代) 初始大小 
>
> -XX:MaxPermSize=N //方法区 (永久代) 最大大小,超过这个值将会抛出 OutOfMemoryError 异常:java.lang.OutOfMemoryError: PermGen
>
> -XX:MetaspaceSize=N //设置 Metaspace 的初始（和最小大小） 
>
> -XX:MaxMetaspaceSize=N //设置 Metaspace 的最大大小，如果不指定大小的话，随着更多类的创建，虚拟机会耗尽所有可用的系统内存。 

### 3.垃圾收集器

> **‐XX:+MaxTenuringThreshold**设置多大年龄的对象进入老年代
>
> -XX:PretenureSizeThreshold（**大对象直接在老年代分配**中的多大的对象）
>
> 

### 收集器指定

>   *   **-XX:+UseSerialGC** :设置串行收集器
>   *   **-XX:+UseParallelGC** :设置并行收集器
>   *   **-XX:+UseParalledlOldGC** :设置并行年老代收集器
>   *   **-XX:+UseConcMarkSweepGC** :设置并发收集器
