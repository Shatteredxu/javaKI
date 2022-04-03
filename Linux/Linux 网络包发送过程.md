## Linux 网络包发送过程

------------------

https://www.eet-china.com/mp/a88353.html

### **1.Linux 网络发送过程总览**

<img src="assets/包发送流程总览.png" alt="img" style="zoom:80%;" />

总体上：用户数据被拷贝到内核态，然后经过协议栈处理后进入到了 RingBuffer 中。随后网卡驱动真正将数据发送了出去。当发送完成的时候，是通过硬中断来通知 CPU，然后清理 RingBuffer。

##### 源码角度的流程图：

![img](assets/包发送流程-源码.png)

##### 包发送结束收尾

![img](assets/包发送结束收尾.png)

### 总结



![img](assets/MBXY-CR-1737b277ee0a16e620c2d820a6cdabb7.png)