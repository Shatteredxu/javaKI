### Spring整体流程

1. 首先从XML或者注解通过<u>BeanDefinitionReader</u>进行解析，最后将这些信息封装在 <u>BeanDefinition</u> 类中，并通过 BeanDefinitionRegistry 接口将这些信息 **注册** 起来，放在 beanDefinitionMap 变量中, 

>    ###### BeanDefinition属性
>
>   *   beanClass : bean 的类型 ，实例化时用的 🐖
>   *   scope : 作用范围有 singleton，prototype
>   *   isLazy : **懒加载** ，true 的话 会在 getBean 时生成，而且 scope 的 prototype 无效，false 在 Spring 启动过程中直接生成
>   *   initMethodName : 初始化方法，当然是初始化时调用 🐖
>   *   primary : 主要的，有多个 Bean 时使用它
>   *   dependsOn : 依赖的 Bean，必须等依赖 Bean 创建好才可以创建



2.有了原料后通过BeanDefinition中的beanClass 利用反射创建对象，放在BeanFactory中，

>   作为 IOC 容器的**根接口** 的 BeanFactory 提供多种方法，以及多个实现类：
>
>   **ListableBeanFactory**
>
>   >   👉 遍历 bean
>
>   **HierarchicalBeanFactory**
>
>   >   👉 提供 父子关系，可以获取上一级的 BeanFactory
>
>   **ConfigurableBeanFactory**
>
>   >   👉 实现了 SingletonBeanRegistry ，主要是 单例 Bean 的注册，生成
>
>   **AutowireCapableBeanFactory**
>
>   >   👉 和自动装配有关的
>
>   **AbstractBeanFactory**
>
>   >   👉 单例缓存，以及 FactoryBean 相关的
>
>   **ConfigurableListableBeanFactory**
>
>   >   👉 预实例化单例 Bean，分析，修改 BeanDefinition
>
>   **AbstractAutowireCapableBeanFactory**
>
>   >   👉 创建 Bean ，属性注入，实例化，调用初始化方法 等等
>
>   **DefaultListableBeanFactory**
>
>   >   👉 支持单例 Bean ，Bean 别名 ，父子 BeanFactory，Bean 类型转化 ，Bean 后置处理，FactoryBean，自动装配等
>
>   

#### ApplicationContext



#### IOC容器

在IOC容器中，你可以对创建出来的Bean进行实例化，初始化等操作，并且在各个过程中合理应用这些 PostProcessor 来扩展，或者修改 Bean 定义信息等等

<img src="assets/image-20211213225748030.png" alt="image-20211213225748030" style="zoom:60%;" />



BeanFactory 后置处理器提供有多个接口：

BeanDefinitionRegistryPostProcessor

>   通过该接口，我们可以自己掌控我们的 **原料**，通过 BeanDefinitionRegistry 接口去 **新增**，**删除**，**获取**我们这个 BeanDefinition

BeanFactoryPostProcessor

>   通过该接口，可以在 **实例化对象前**，对 BeanDefinition 进行修改 ，**冻结** ，**预实例化单例 Bean** 等

##### Bean生命周期

经过上面的接口后，我们会将一个完整的Bean创建出来，随之而来的就是这个 Bean 的生命周期，一共有14个步骤

![image-20211213225831583](assets/image-20211213225831583.png)

里面包括**实例化** 👉 **Instantiation**，**初始化** 👉 **Initialization**，而xxAware 意思就是获取这个单词的前缀 xx ，

##### Bean 后置处理器

![image-20211213225953964](assets/image-20211213225953964.png)

##### AOP

![image-20211213230042502](assets/image-20211213230042502.png)

##### 总结

主要介绍了 Spring 里面的这些脉络，方便小伙伴们对它有个整体的印象先~

再介绍其中的一些扩展点，比如从源材料开始的 BeanFactoryPostprocessor ，到产物 Bean 的 BeanPostprocessor 。

实例化，初始化的顺序，Bean 的生命周期，以及 BeanFactory 及子类扩展的功能，再到 ApplicationContext 的功能。

![image-20211213230212297](assets/image-20211213230212297.png)
