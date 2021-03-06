Etcd原理详解
------------

ETCD是用于共享配置和服务发现的分布式，一致性的KV存储系统。基于Go语言实现，主要用于<u>共享配置</u>，<u>服务发现</u>，<u>集群监控</u>，<u>Leader选举</u>，<u>分布式锁</u>等场景。

ETCD集群是一个分布式系统，由多个节点相互通信构成整体对外服务，每个节点都存储了完整的数据，并且通过**Raft协议**保证每个节点维护的数据是一致的。

### 1.选主：Raft

### 2.日志复制

日志复制是指主节点将每次操作形成日志条目，并持久化到本地磁盘，然后通过网络IO发送给其他节点。其他节点根据日志的<u>逻辑时钟(TERM)和日志编号</u>(INDEX)来判断是否将该日志记录持久化到本地。当主节点收到包括自己在内超过半数节点成功返回，那么认为该日志是可提交的(committed），并将日志输入到状态机，将结果返回给客户端。

主节点通过网络IO向其他节点追加日志。若某节点收到日志追加的消息，首先判断该日志的TERM是否过期，以及该日志条目的INDEX是否比当前以及提交的日志的INDEX跟早。若已过期，或者比提交的日志更早，那么就拒绝追加，并返回该节点当前的已提交的日志的编号。否则，将日志追加，并返回成功。

### 与其他数据库对比

##### 与Zookeeper的不同

Zookeeper和etcd解决的问题是一样的，都解决分布式系统的协调和元数据的存储，所以它们都不是一个存储组件，或者说都不是一个分布式数据库。但是etcd相对于zookeeper做了一些改进：

*   更轻量级、更易用
*   etcd有分布式存储功能，对于大数据的存储写入性能会比zk要好很多
*   数据模型的多版本并发控制
*   watch机制方main更加优秀
*   客户端协议使用gRPC协议，支持go、C++、Java等，而Zookeeper的RPC协议是自定制的，目前只支持C和Java
*   可以容忍脑裂现象的发生

##### 与Redis对比

1.   Etcd的最初来源于k8s用etcd做服务发现，而redis的则来源于memcache缓存本身的局限性。
2.   etcd的重点是利用raft算法做分布式一致性，强调各个节点之间的通信、同步，确保各节点数据和事务的一致性；
3.   etcd v3的底层采用boltdb做存储，value直接持久化；redis是一个内存数据库，它的持久化方案有aof和rdb，在宕机时都或多或少会丢失数据。
4.   etcd v3只能通过gRPC访问，而redis可以通过http访问，因此etcd的客户端开发工作量高很多。
5.   redis的性能比etcd强。
6.   redis的value支持多种数据类型。
7.   Redis注重存取数据，Etcd注重服务间节点一致性

##### 与Consul，euerka对比

| Feature              | Consul                        | **zookeeper**                  | etcd              | euerka                       | Nacos                      |
| -------------------- | ----------------------------- | ------------------------------ | ----------------- | ---------------------------- | -------------------------- |
| 服务健康检查         | TCP/HTTP/gRPC/Cmd             | (弱)长连接，keepalive          | 连接心跳          | 可配支持                     | TCP/HTTP/MYSQL/Client Beat |
| 多数据中心           | 支持                          | —                              | —                 | —                            |                            |
| kv存储服务           | 支持                          | 支持                           | 支持              | —                            |                            |
| 一致性               | raft单向通信,平时同步时会回溯 | zab新leader需要数据同步,拉数据 | raft              | —                            |                            |
| cap                  | **cp**                        | cp                             | cp                | ap适合服务发现               | AP+CP模式混合              |
| 使用接口(多语言能力) | 支持http和dns                 | 客户端                         | http/grpc         | http（sidecar）              |                            |
| watch支持            | 全量/支持long polling         | 支持                           | 支持 long polling | 支持 long polling/大部分增量 |                            |
| 自身监控             | metrics                       | —                              | metrics           | metrics                      |                            |
| 安全                 | acl /https                    | acl                            | https支持（弱）   | —                            |                            |
| spring cloud集成     | 已支持                        | 已支持                         | 已支持            | 已支持                       |                            |



