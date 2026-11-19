---
title: ACID 实现原理
---

# ACID 实现原理

官方说明：ACID 是一组强调数据可靠性、对业务数据与关键任务应用很重要的数据库设计原则。InnoDB 存储引擎**紧密遵循 ACID 模型**，使数据在软件崩溃、硬件故障等异常情况下不会被破坏、结果不会被扭曲。如果应用能容忍少量数据丢失或不一致，也可以调整设置，用部分 ACID 可靠性换取更高性能。

下面分别说明原子性（A）、一致性（C）、隔离性（I）、持久性（D）的实现思路，并链接站内相关文档。

## 原子性（Atomicity）

**关注点**：事务的 `COMMIT` 与 `ROLLBACK`。

官方说明：原子性主要涉及 InnoDB 事务。一个事务中的操作要么全部提交，要么全部回滚。

**实现方式**

- 事务是原子性的基本单位：以 `START TRANSACTION` 开始，以 `COMMIT` 提交或 `ROLLBACK` 回滚。
- **undo log** 记录数据修改前的状态：事务回滚时，依据 undo log 把数据恢复到事务开始前的状态。
- 相关文档：[Undo Log](../logs/undo-log.md)、[事务概览](index.md)。

## 一致性（Consistency）

**关注点**： InnoDB 内部为保护数据免受崩溃影响的处理。

官方说明：一致性主要涉及 InnoDB 内部为保护数据免受崩溃影响的处理，明确列出 **doublewrite buffer** 与 **crash recovery**。

**实现方式**

- 一致性是事务追求的最终结果：事务执行前后，数据都必须满足既定的完整性约束（主键、唯一、外键等）。
- **doublewrite buffer**：防止部分页写入（partial page write）导致的数据页损坏。
- **crash recovery**：数据库异常重启后，通过重做已提交事务、回滚未提交事务，恢复到一致状态。
- 相关文档：[Redo Log](../logs/redo-log.md)、[Checkpoint](../terms/checkpoint.md)、[LSN](../terms/lsn.md)、[Consistent Read](../terms/consistent-read.md)。

## 隔离性（Isolation）

**关注点**：事务隔离级别，每个事务适用的隔离程度。

官方说明：隔离性主要涉及 InnoDB 事务，尤其是每个事务适用的隔离级别（`SET TRANSACTION`）；底层细节是 InnoDB locking，可通过 `information_schema` 的 `data_locks`、`data_lock_waits` 查看锁信息。官方下一节即为 InnoDB Multi-Versioning。

**实现方式**

- **锁机制**：行锁、间隙锁、临键锁等控制并发写与范围插入，见 [锁机制](../concurrency/locking.md)。
- **MVCC / ReadView**：一致性非锁定读通过版本链与 ReadView 读取快照，实现读写不冲突，见 [MVCC & ReadView](../concurrency/mvcc-readview.md) 与 [Multi-Versioning](../storage-engine/innodb-multi-versioning.md)。
- **隔离级别**：`READ UNCOMMITTED`、`READ COMMITTED`、`REPEATABLE READ`、`SERIALIZABLE` 在一致性与并发性能之间取舍，见 [事务隔离级别](../concurrency/transaction-isolation.md) 与 [并发事务问题](../concurrency/multi-transaction-concurrency.md)。

## 持久性（Durability）

**关注点**：软件特性与硬件配置的交互，这是 ACID 中最复杂的一项。

官方说明：持久性涉及 MySQL 软件特性与具体硬件配置的交互，因为 CPU、网络、存储设备的差异很大，有时官方建议甚至可能是“购买新硬件”。

**实现方式**

- **redo log**：事务提交时写入 redo log，通过 `innodb_flush_log_at_trx_commit` 控制刷盘策略，保证已提交事务的修改在崩溃后可恢复，见 [Redo Log](../logs/redo-log.md)。
- **binary log**：通过 `sync_binlog` 控制刷盘，配合 redo log 保证复制与时间点恢复的数据不丢失，见 [Binary Log](../logs/binary-log.md)。
- **doublewrite buffer**：避免页写入过程中宕机造成的数据页损坏。
- **硬件与系统**：存储设备写缓存与电池备份缓存、操作系统 `fsync()` 支持、UPS 与备份策略等。
- 相关文档：[MySQL 日志详解](../logs/mysql-logs-overview.md)、[Checkpoint](../terms/checkpoint.md)。

## 对照总结

| 特性 | 核心机制 | 站内文档 |
|---|---|---|
| A 原子性 | 事务、`COMMIT` / `ROLLBACK`、undo log | [Undo Log](../logs/undo-log.md) |
| C 一致性 | 完整性约束、doublewrite buffer、crash recovery | [Redo Log](../logs/redo-log.md)、[Checkpoint](../terms/checkpoint.md) |
| I 隔离性 | 隔离级别、锁、MVCC / ReadView | [事务隔离级别](../concurrency/transaction-isolation.md)、[锁机制](../concurrency/locking.md)、[MVCC & ReadView](../concurrency/mvcc-readview.md) |
| D 持久性 | redo log、binlog 刷盘、doublewrite buffer、硬件与备份 | [Redo Log](../logs/redo-log.md)、[Binary Log](../logs/binary-log.md) |

## 参考

- [MySQL 8.0 Reference Manual · 17.2 InnoDB and the ACID Model](https://dev.mysql.com/doc/refman/8.0/en/mysql-acid.html)
- [MySQL 8.0 Reference Manual · 17.7.1 InnoDB Locking](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)
- [MySQL 8.0 Reference Manual · 10.3.1 How MySQL Uses Indexes](https://dev.mysql.com/doc/refman/8.0/en/mysql-indexes.html)
