# MySQL

本部分按“从连接到执行、从逻辑到物理、再到部署”的脉络组织：先了解整体架构与连接管理，再沿 SQL 执行链路理解查询解析、优化与索引，随后深入事务、并发控制与存储层，最后覆盖日志、常用术语与高可用集群。

## 整体架构

- [MySQL 架构总览](mysql-architecture.md)

## 连接与线程管理

连接建立与线程处理是 SQL 执行的起点，涵盖服务端线程模型与应用侧连接复用。

- [概览](connection-management/index.md)
- [连接池](connection-management/connection-pool.md)
- [Thread Handler](connection-management/thread-handler.md)

## 查询解析与优化

一条 SQL 从文本到结果，依次经过解析、优化与执行，执行计划是观察这一过程的窗口。

- [概览](query-optimization/index.md)
- [查询解析器](query-optimization/query-parser.md)
- [优化器](query-optimization/optimizer.md)
- [执行计划](query-optimization/execution-plan.md)
- 番外篇：[index Cardinality](query-optimization/bonus-cardinality.md)

## 存储引擎

MySQL 使用可插拔存储引擎架构，重点介绍默认的 InnoDB，并对比 MyISAM。

- [概览](storage-engine/index.md)
- [InnoDB Overview](storage-engine/innodb-overview.md)
- [InnoDB Architecture](storage-engine/innodb-architecture.md)
- [In-Memory Structures](storage-engine/innodb-in-memory-structures.md)
- [Multi-Versioning](storage-engine/innodb-multi-versioning.md)
- [MyISAM](storage-engine/myisam.md)

## 索引

索引是查询加速的核心，先理解数据结构，再学习如何设计索引。

- [概览](index-structure/index.md)
- [索引数据结构](index-structure/index-data-structure.md)
- [索引设计](index-structure/index-design.md)
- 番外篇：[索引长度计算](index-structure/bonus-index-key-len.md)、[页分裂](index-structure/bonus-page-split.md)

## 事务

事务的 ACID 特性及其实现机制。

- [概览](transaction/index.md)
- [ACID 实现原理](transaction/acid.md)
- 番外篇：[悬挂事务](transaction/bonus-dangling-transaction.md)

## 并发控制

事务隔离、MVCC 与锁机制，共同支撑高并发下的一致性与性能。

- [概览](concurrency/index.md)
- [事务隔离级别](concurrency/transaction-isolation.md)
- [并发事务问题](concurrency/multi-transaction-concurrency.md)
- [MVCC & ReadView](concurrency/mvcc-readview.md)
- [锁机制](concurrency/locking.md)
- 示例：[RC 只锁修改的行](concurrency/example-rc-locks.md)、[分析不同的隔离级别](concurrency/example-isolation-levels.md)

## 日志

二进制日志与 InnoDB 的 redo/undo 日志，支撑复制、崩溃恢复与事务。

- [概览](logs/index.md)
- [MySQL 日志详解](logs/mysql-logs-overview.md)
- [Binary Log](logs/binary-log.md)
- [Redo Log](logs/redo-log.md)
- [Undo Log](logs/undo-log.md)

## Terms

常见术语速查。

- [概览](terms/index.md)
- [Checkpoint](terms/checkpoint.md)
- [LSN](terms/lsn.md)
- [Consistent Read](terms/consistent-read.md)

## 高可用与集群

从主从复制到组复制与 InnoDB Cluster，构建高可用架构。

- [概览](high-availability/index.md)
- [MySQL 主从复制](high-availability/master-slave-replication.md)
- [MGR、InnoDB Cluster 与自动切换](high-availability/mgr-innodb-cluster.md)
