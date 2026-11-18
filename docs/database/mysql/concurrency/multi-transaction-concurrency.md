---
title: 并发事务问题
---

# 并发事务问题

MySQL 服务器支持多个客户端同时连接，同一时刻可能存在多个事务并发执行。与单线程处理不同，并发事务之间会相互干扰，带来三类典型的读现象：**脏读（dirty read）**、**不可重复读（non-repeatable read）**、**幻读（phantom）**。

## 三种读现象

### 脏读（Dirty Read）

> MySQL 8.0 Glossary：
> "An operation that retrieves unreliable data, data that was updated by another transaction but not yet committed. It is only possible with the isolation level known as read uncommitted."

即读取到被其他事务修改但**尚未提交**的数据。官方强调这类数据是不可靠的（unreliable）：原事务可能回滚，也可能在提交前继续修改，读取方拿到的是从未被确认过的结果。

例：事务 B 将余额从 100 改为 200 但未提交；事务 A 读到 200；随后 B 回滚，A 读到的 200 从未真正生效过。

只在 `READ UNCOMMITTED` 隔离级别下发生。

### 不可重复读（Non-repeatable Read）

> MySQL 8.0 Glossary：
> "The situation when a query retrieves data, and a later query within the same transaction retrieves what should be the same data, but the queries return different results (changed by another transaction committing in the meantime)."

即同一事务中，后一次查询本应得到与前一次相同的数据，却因为另一事务在此期间提交而返回了不同结果。

例：事务 A 读取某行余额为 100；事务 B 将其改为 200 并提交；事务 A 再次读取同一行得到 200。

侧重点是**同一行数据的内容变化**（`UPDATE` / `DELETE`）。

### 幻读（Phantom）

> MySQL 8.0 Glossary：
> "A row that appears in the result set of a query, but not in the result set of an earlier query. For example, if a query is run twice within a transaction, and in the meantime, another transaction commits after inserting a new row or updating a row so that it qualifies for the WHERE clause of the second query."

即后一次查询的结果集中出现了前一次没有的行。除了 `INSERT` 新行，另一事务的 `UPDATE` 使某行变得满足 `WHERE` 条件，同样会产生幻读。

例：事务 A 查询 `WHERE age > 20` 得到 3 行；事务 B 插入一条 `age = 25` 的记录并提交；事务 A 再次执行同样的查询得到 4 行。

侧重点是**结果集成员的变化**，而非某一行本身被改写。

### 小结

| 现象 | 关注点 | 典型成因 | 发生条件 |
|---|---|---|---|
| 脏读 | 读到未提交的数据 | 其他事务的未提交修改 | `READ UNCOMMITTED` |
| 不可重复读 | 同一行前后值不同 | 其他事务提交 `UPDATE` / `DELETE` | `READ UNCOMMITTED`、`READ COMMITTED` |
| 幻读 | 结果集出现新行 | 其他事务提交 `INSERT`，或 `UPDATE` 使行满足 `WHERE` | `READ UNCOMMITTED`、`READ COMMITTED`（RR 下见下文） |

## 隔离级别如何避免这些问题

SQL 标准定义了四种隔离级别，级别越低并发性能越好，但可容忍的异常越多。InnoDB 支持全部四种，默认是 `REPEATABLE READ`。

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | InnoDB 实现要点 |
|---|---|---|---|---|
| `READ UNCOMMITTED` | 允许 | 允许 | 允许 | 普通 `SELECT` 非锁定，可能读到行的较早版本 |
| `READ COMMITTED` | 避免 | 允许 | 允许 | 每次一致性读新建 ReadView；锁定读只锁索引记录，不锁 gap |
| `REPEATABLE READ`（默认） | 避免 | 避免 | 快照读由 MVCC 保证；当前读由 gap / next-key lock 阻止向范围插入 | 同一事务的多次一致性读复用第一次建立的快照 |
| `SERIALIZABLE` | 避免 | 避免 | 避免 | `autocommit=0` 时普通 `SELECT` 隐式转为 `SELECT ... FOR SHARE`；`autocommit=1` 时 `SELECT` 是独立只读事务，可作一致性非锁定读 |

各隔离级别的加锁细节与示例见 [事务隔离级别](transaction-isolation.md)。

## InnoDB 实现视角：快照读与当前读

三类现象既取决于隔离级别，也取决于读的方式。InnoDB 中有两种读：

- **快照读（一致性非锁定读）**：普通 `SELECT`，基于 MVCC。事务通过 ReadView 从 `undo log` 的版本链中筛选出可见版本，**不加锁**。
    - `READ COMMITTED`：每次一致性读都创建新的 ReadView，因此同一事务内两次查询可能看到不同结果。
    - `REPEATABLE READ`：第一次一致性读创建 ReadView，整个事务沿用（`START TRANSACTION WITH CONSISTENT SNAPSHOT` 可在事务开始时就建立快照），因此多次查询结果一致。
- **当前读（锁定读）**：`SELECT ... FOR UPDATE`、`SELECT ... FOR SHARE`、`UPDATE`、`DELETE`，读取**最新版本并加锁**。`REPEATABLE READ` 下对范围扫描使用 gap lock / next-key lock，阻止其他事务向已锁范围插入，从而避免当前读的幻读。

!!! warning "不要混用快照读与当前读"
    InnoDB 官方文档明确不建议在同一个 `REPEATABLE READ` 事务中混用普通 `SELECT` 与锁定语句（`UPDATE`、`INSERT`、`DELETE`、`SELECT ... FOR ...`）：普通 `SELECT` 看到的是 ReadView 中的快照状态，锁定语句使用的是数据库最新状态并加锁，两者通常不一致。这正是“快照读与当前读混用导致幻读”的根源。

MVCC、ReadView 与两种读的关系见 [MVCC & ReadView](mvcc-readview.md)。

## 参考

- [MySQL 8.0 Glossary](https://dev.mysql.com/doc/refman/8.0/en/glossary.html)
- [MySQL 8.0 Reference Manual · 17.7.2.1 Transaction Isolation Levels](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html)
