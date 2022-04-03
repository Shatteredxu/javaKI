#### @SPI 注解

Dubbo 中某个接口被 @SPI注解修饰时，就表示该接口是**扩展接口**，前文示例中的 org.apache.dubbo.rpc.Protocol 接口就是一个扩展接口：

#### @Adaptive 注解与适配器

**AdaptiveExtensionFactory 不实现任何具体的功能，而是用来适配 ExtensionFactory 的 SpiExtensionFactory 和 SpringExtensionFactory 这两种实现。AdaptiveExtensionFactory 会根据运行时的一些状态来选择具体调用 ExtensionFactory 的哪个实现。**

#### **自动包装特性****Wrapper**

**@Activate 注解**