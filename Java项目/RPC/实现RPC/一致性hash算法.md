一致性哈希算法
--------------

### 原理

首先我们要了解<u>传统的哈希及其在大规模分布式系统中的局限性</u>。简单地说，哈希就是一个键值对存储，在给定键的情况下，可以非常高效地找到所关联的值。在分布式负载均衡场景中，我们需要根据客户端请求，给客户端分配一个后端服务器进行访问，通常情况下我们会采用**hash算法取模**来寻找一个服务器来进行访问，但是使用传统hash有诸多不便：1.节点减少情况（节点的减少会导致键与节点的映射关系发生变化，对于已有的键来说，将导致节点映射错误）2. 节点增加（会导致映射错误如果集群中每个机器提供的服务没有差别，这不会有什么影响。**但对于分布式缓存这种的系统而言，映射规则失效就意味着之前缓存的失效，若同一时刻出现大量的缓存失效，则可能会出现 “缓存雪崩”，这将会造成灾难性的后果。**）

##### 一致性hash算法

1.   一致性哈希算法通过一个叫作一致性哈希环的数据结构实现。这个环的起点是 0，终点是 2^32 - 1，并且起点与终点连接，故这个环的整数分布范围是 [0, 2^32-1]
2.   将服务器放置到哈希环
3.   假设我们有 "semlinker"、"kakuqo"、"lolo"、"fer" 四个对象，分别简写为 o1、o2、o3 和 o4，然后使用哈希函数计算这个对象的 hash 值，值的范围是 [0, 2^32-1]：
4.   寻找距离对象hash最近的服务器
5.   当我们增加一台服务器，新增服务器只会影响这台新增服务器及其后面的一个服务器，所以只有少部分对象会有影响，减少一台服务器同理
6.   还**可以通过引入虚拟节点来解决负载不均衡的问题。即将每台物理服务器虚拟为一组虚拟服务器，将虚拟服务器放置到哈希环上，如果要确定对象的服务器，需先确定对象的虚拟服务器，再由虚拟服务器确定物理服务器。**有虚拟节点映射回物理服务器

### min-RPC实现

1.   rpc-registry中的RegistryService中定义注册中心的四种方法（register，unRegister，discovery，destroy）

2.   ZookeeperRegistryService中实现RegistryService并且实现其四种方法，初始化传入注册中心地址，设置重试时间，重试次数

3.   使用Apache Curator 初始化 Zookeeeper 客户端

     >   Apache Curator 是一个比较完善的ZooKeeper客户端框架，通过封装的一套高级API 简化了ZooKeeper的操作，提供了重试机制，连接状态监控功能，创建持久化节点时候递归创建
     >
     >   ExponentialBackoffRetry：指数补偿算法

4.   使用TreeMap来存放真实的节点和虚拟的节点，key为hash，value为服务器信息
5.   在discovery方法里面会先获取所有的节点，然后找到大于或等于给定键元素(ele)的最小键元素链接的键值对（ceilingEntry）方法或者tailMap，再判断节点是否存在，如果不存在说明是最后一个则返回第一个节点

### Hash算法

MD4,MD5，**SHA-1**

```java
//MD5算法的实现
public class HashFunction {
    private MessageDigest md5 = null;

    public long hash(String key) {
        if (md5 == null) {
            try {
                md5 = MessageDigest.getInstance("MD5");
            } catch (NoSuchAlgorithmException e) {
                throw new IllegalStateException("no md5 algrithm found");
            }
        }

        md5.reset();
        md5.update(key.getBytes());
        byte[] bKey = md5.digest();
        //具体的哈希函数实现细节--每个字节 & 0xFF 再移位
        long result = ((long) (bKey[3] & 0xFF) << 24)
                | ((long) (bKey[2] & 0xFF) << 16
                | ((long) (bKey[1] & 0xFF) << 8) | (long) (bKey[0] & 0xFF));
        return result & 0xffffffffL;
    }
}
```

哈希算法特点：

1.  正像快速：原始数据可以快速计算出哈希值
2.  逆向困难：通过哈希值基本不可能推导出原始数据
3.  输入敏感：原始数据只要有一点变动，得到的哈希值差别很大
4.  冲突避免：很难找到不同的原始数据得到相同的哈希值

MD5具体流程：

1.   数据填充，使消息的长度X对512取模得448
2.   在第一步结果之后再填充上原消息的长度，可用来进行的存储长度为64位。如果**消息长度大于264，则只使用其低64位的值**，即（消息长度 对 264取模）。
3.   把消息经过四组计算得到最终的

