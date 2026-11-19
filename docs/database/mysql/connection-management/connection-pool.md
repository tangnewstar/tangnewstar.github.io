---
title: 连接池
---

# 连接池

连接池（Connection Pool）是应用侧维护的一组可复用的数据库连接。应用启动或首次使用时预先建立若干连接，业务使用时从池中借出、用完后归还，而不是每次请求都向 MySQL 新建连接、用完再断开。

连接池要解决的根本问题，来自 MySQL 的连接模型。官方说明：MySQL 把每个客户端连接关联到一个**专用线程**负责认证与处理请求；连接数很大时，线程创建销毁成本、每线程内存（栈空间）、调度开销都会变得明显（见 [Thread Handler](thread-handler.md)）。因此，**在服务端线程模型之上，应用侧复用连接是降低连接成本的主要手段**。

## 为什么需要连接池

- **避免反复建连/断连的开销**：TCP 三次握手、MySQL 认证、线程分配都需要成本；连接结束后线程若不能复用，还要销毁。
- **控制并发连接数**：MySQL 用 `max_connections` 限制最大客户端连接数，超限时连接被拒绝，`Connection_errors_max_connections` 递增。连接池把应用侧并发连接数约束在一个可控范围。
- **降低服务端线程压力**：配合服务端的 `thread_cache_size`（线程缓存），减少 `Threads_created` 的持续增长。

!!! note "连接池与服务端线程的关系"
    应用连接池解决的是“连接复用”，服务端线程池（MySQL Enterprise Thread Pool）解决的是“线程复用”，两者在不同层面降低开销，并不互相替代。

## 连接池大小：不是越大越好

一个常见误区是把连接池开到尽可能大。成熟连接池社区（HikariCP）的核心观点是：**连接池应该足够小，小到能 saturate CPU 即可**。原因与 Little's Law 有关——连接数（线程数）超过某个点后，增加连接只会让更多线程排队等待 CPU 和锁，上下文切换开销上升，吞吐不升反降。

HikariCP 社区给出的经验公式（源自 PostgreSQL 经验、被广泛引用）：

```text
连接数 = (CPU 核心数 × 2) + 有效磁盘数
```

例如 4 核 CPU、1 块 SSD，连接池大小约 `4×2 + 1 = 9`。这只是起点，实际应结合负载类型（CPU 密集还是 I/O 密集）、事务平均耗时与期望响应时间，通过压测调整。

要点：

- 连接池过大 → 更多活跃事务、更多锁竞争、更高上下文切换成本。
- 连接池过小 → 请求排队等待连接， latency 增加。
- 目标是让连接数与 CPU、磁盘处理能力匹配，而不是与并发请求数匹配。

## 关键配置项

主流连接池（HikariCP、Druid、Tomcat JDBC 等）配置项大同小异，可归纳为几类：

### 1. 池大小

| 参数（HikariCP 为例） | 含义 |
|---|---|
| `maximumPoolSize` | 池中最大连接数，核心参数，按上面的公式估算 |
| `minimumIdle` | 池中保持的最小空闲连接数 |

### 2. 超时与生命周期

| 参数 | 含义 |
|---|---|
| `connectionTimeout` | 从池中借连接的最大等待时间，超时抛异常 |
| `idleTimeout` | 空闲连接被回收前的空闲时长（仅在连接数大于 `minimumIdle` 时生效） |
| `maxLifetime` | 连接的最大存活时间；应略小于 MySQL 的 `wait_timeout`，避免使用已被服务端关闭的连接 |
| `validationTimeout` | 连接有效性校验的超时时间 |

### 3. 连接保活与校验

- 定期用轻量 SQL（如 `SELECT 1`）校验连接有效性，避免使用被服务端因超时断开的“死连接”。
- `maxLifetime` 应小于 MySQL 的 `wait_timeout` / `interactive_timeout`，这是生产环境最常见的配置要点之一。

## 与 MySQL 驱动配合的优化

HikariCP 官方 wiki 针对 MySQL / Connector-J 给出一组建议，重点是**预处理语句缓存**（注意属性名是 `prepStmtCacheSize`）：

```properties
dataSource.cachePrepStmts=true           # 开启 prepared statement 缓存，否则下面的缓存参数不生效
dataSource.prepStmtCacheSize=250         # 每连接缓存的 prepared statement 数，建议 250-500（默认仅 25）
dataSource.prepStmtCacheSqlLimit=2048    # 可缓存 SQL 的最大长度，默认 256 对 ORM 生成的 SQL 偏小
dataSource.useServerPrepStmts=true       # 使用服务端预处理语句，可显著减少重复解析开销
dataSource.useLocalSessionState=true
dataSource.rewriteBatchedStatements=true
dataSource.cacheResultSetMetadata=true
dataSource.cacheServerConfiguration=true
dataSource.elideSetAutoCommits=true
dataSource.maintainTimeStats=false
```

这些设置的收益可从 MySQL 官方文档得到印证：服务端预处理语句利用二进制协议，**减少每次执行时解析语句的开销**，并降低 SQL 注入风险（见 [查询解析器](../query-optimization/query-parser.md)）。

## 常见实践

- **应用启动时预热**：初始化时建立最小空闲连接，避免首批请求集中建连。
- **借出/归还规范**：务必在 `finally` 或 `try-with-resources` 中归还连接，避免连接泄漏；连接泄漏会迅速耗尽池。
- **监控**：关注池的活跃连接数、等待借连接的时间、连接创建速率；结合 MySQL 的 `Threads_connected`、`Threads_created`、`max_connections` 使用情况判断瓶颈在应用侧还是服务端。
- **与超时配合**：池的连接超时、事务超时应与 MySQL 的 `innodb_lock_wait_timeout`、`wait_timeout` 协调，避免层层超时时间错配。

## 相关文档

- [Thread Handler](thread-handler.md)
- [查询解析器](../query-optimization/query-parser.md)

## 参考

- [MySQL 8.0 Reference Manual · 7.1.12.1 Connection Interfaces](https://dev.mysql.com/doc/refman/8.0/en/connection-interfaces.html)
- [MySQL 8.0 Reference Manual · 15.5 Prepared Statements](https://dev.mysql.com/doc/refman/8.0/en/sql-prepared-statements.html)
- [HikariCP Wiki · MySQL Configuration](https://github.com/brettwooldridge/HikariCP/wiki/MySQL-Configuration)
- [HikariCP Wiki · About Pool Sizing](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing)
