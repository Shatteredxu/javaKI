### 1. Redis源码整体架构

那么对于 Redis 来说，在它的源码总目录下，一共包含了`deps`、`src`、`tests`、 `utils`四个子目录。

`deps`包含了 Redis 依赖的第三方代码库，包括 Redis 的 C 语言版本客户端代码 hiredis、jemalloc 内存分配器代码、readline 功能的替代代码 linenoise，以及 lua 脚本代码。

`src`包含了 Redis 所有功能模块的代码文件，也是 Redis 源码的重要组成部分

**`tests`** 单元测试（对应 unit 子目录），RedisCluster 功能测试（对应 cluster 子目录）、哨兵功能测试（对应 sentinel 子目录）、主**从复制功能测试（对应 integration 子目录）

`utils`辅助性功能，包括用于创建 Redis Cluster 的脚本、用于测试 LRU 算法效果的程序，以及可视化 rehash 过程的程序。

