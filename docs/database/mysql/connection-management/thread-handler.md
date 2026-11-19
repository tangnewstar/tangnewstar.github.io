---
title: Thread Handler
---

# Thread Handler

Thread Handler 指 MySQL Server 处理客户端连接与语句执行的线程模型。官方在连接管理章节中说明：MySQL 通过**连接处理线程（connection handler threads）**管理连接，连接接口收到请求后，为每个客户端连接关联一个专用线程，负责认证与处理该连接上的请求（见 [Connection Interfaces](https://dev.mysql.com/doc/refman/8.0/en/connection-interfaces.html)）。

理解 Thread Handler，关键是理解 MySQL 默认的“每连接一线程”模型及其替代方案（线程池），以及它们与 `thread_handling`、`thread_cache_size` 等参数的关系。

## 连接处理流程

```text
客户端连接请求
  → 连接管理线程（监听 TCP/IP、Unix socket 等接口）
  → 检查线程缓存，有可用线程则复用，否则创建新线程
  → 认证
  → 该线程处理连接上的所有请求（会话级）
  → 连接结束，线程放回缓存（若缓存未满）或销毁
```

官方说明：单个服务器插件也能实现自己的连接接口，例如 X Plugin 让 MySQL 通过 X Protocol 与客户端通信。

## 默认模型：one-thread-per-connection

`thread_handling` 系统变量表示服务器使用的线程处理模型，默认值为 `one-thread-per-connection`：

| 属性 | 值 |
|---|---|
| 命令行格式 | `--thread-handling=name` |
| 作用域 | Global |
| 是否动态修改 | 否 |
| 默认值 | `one-thread-per-connection` |
| 有效取值 | `one-thread-per-connection`、`loaded-dynamically` |

官方指出的缺点：这种模型下，当前连接的客户端数量基本等于服务器中的线程数量。当连接数很大时：

- 线程创建和销毁成本变高；
- 每个线程都需要服务器和内核资源，例如**栈空间**（`thread_stack`）；
- 为支持大量并发连接往往要减小每线程栈大小，但这会限制复杂 SQL、存储过程递归深度；
- 可能出现其他资源耗尽；
- 调度开销变得明显。

## 线程缓存：thread_cache_size

为缓解线程频繁创建销毁，MySQL 提供线程缓存。官方说明：

- `thread_cache_size` 决定缓存的线程数，默认 `-1`（自动调整），范围 `0~16384`，设为 `0` 表示禁用。
- 客户端断开时，若缓存未满，线程被放入缓存；新连接到来时优先复用缓存线程；没有可用缓存线程才创建新线程；缓存中超量的线程会被终止。
- 主要用于提升**频繁连接和断开**场景的性能。

相关状态变量：

| 变量 | 含义 |
|---|---|
| `Threads_cached` | 当前缓存中的线程数 |
| `Threads_created` | 因缓存无可用线程而新建的线程数，持续增长说明创建频繁，可考虑调大缓存 |
| `Threads_connected` | 当前连接数 |

!!! note "连接数与线程数的关系"
    在默认模型下，`Threads_connected` 基本等于服务器中的工作线程数；而 `Threads_created` 反映累计创建线程次数。两者配合 `thread_cache_size` 一起观察，可判断线程是否稳定复用。

## 企业版线程池：loaded-dynamically

当服务器成功加载线程池插件时，`thread_handling` 变为 `loaded-dynamically`。这是 MySQL Enterprise Edition 的商业功能，通过服务器插件提供替代的线程模型。

官方说明：默认模型在连接数和并发语句数增加时性能下降；线程池通过以下方式更高效地管理语句执行线程：

1. **复用线程栈**：减少线程栈数量，降低高并行负载下的 CPU cache footprint。
2. **控制活跃线程数量**：避免过多线程并行导致上下文切换和操作系统调度器压力，把 MySQL 内部并行度控制在服务器可处理的范围。
3. **控制事务启动时机**：避免过多事务同时执行导致的资源竞争，尤其是在 InnoDB 中减少持有中心 mutex 的时间。

社区版 MySQL 不提供该插件；大量连接场景通常通过应用侧[连接池](connection-pool.md)、合理的 `max_connections` 与线程缓存来应对。

| 对比项 | one-thread-per-connection | 线程池（Thread Pool） |
|---|---|---|
| 线程模型 | 每连接一个线程 | 多连接共享受控工作线程 |
| 适用 | 连接数较少、并发较低 | 大量连接、高并发 |
| CPU 缓存 | 线程栈多，高并行下缓存效果下降 | 复用线程栈，缓存压力小 |
| 上下文切换 | 线程多，切换开销高 | 控制活跃线程数，降低调度压力 |
| 事务并发 | 并行事务多，mutex 争用高 | 控制事务启动，降低争用 |
| 版本 | 社区版/企业版默认 | MySQL Enterprise Edition 插件 |

## 连接数量管理

官方用 `max_connections` 控制服务器允许的最大客户端连接数：

- 超过该值时连接被拒绝，状态变量 `Connection_errors_max_connections` 递增；
- `mysqld` 实际允许的连接数是 `max_connections + 1`，额外一个保留给具有 `CONNECTION_ADMIN`（或已弃用的 `SUPER`）权限的账户，便于管理员在连接占满时仍能连入诊断（如 `SHOW PROCESSLIST`）；
- MySQL 8.0.14 起支持管理网络接口，可使用专用 IP 和端口接受管理连接。

官方指出，最大可支持的连接数取决于：平台线程库质量、可用内存、每连接内存与工作负载、期望响应时间、可用文件描述符数量。参考值：Linux/Solaris 通常可稳定支持至少 500~1000 并发连接；内存数 GB 且每连接负载较低时可达 10000。提高 `max_connections` 会增加文件描述符需求，必要时需提高 `open_files_limit`。

## 与其他模块的联系

- **连接池**：应用侧复用连接，减少新建连接与服务端的线程创建销毁（见 [连接池](connection-pool.md)）。
- **锁与事务**：线程是事务执行的载体，过多并行事务会加剧锁竞争与 InnoDB mutex 争用（见 [锁机制](../concurrency/locking.md)）。
- **查询解析**：每个连接上的语句都要经过解析、优化、执行（见 [查询解析器](../query-optimization/query-parser.md)）。

## 参考

- [MySQL 8.0 Reference Manual · 7.1.12 Connection Management](https://dev.mysql.com/doc/refman/8.0/en/connection-management.html)
- [MySQL 8.0 Reference Manual · 7.1.12.1 Connection Interfaces](https://dev.mysql.com/doc/refman/8.0/en/connection-interfaces.html)
- [MySQL 8.0 Reference Manual · 7.6.3 MySQL Enterprise Thread Pool](https://dev.mysql.com/doc/refman/8.0/en/thread-pool.html)
- [MySQL 8.0 Reference Manual · 7.1.8 Server System Variables](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html)
