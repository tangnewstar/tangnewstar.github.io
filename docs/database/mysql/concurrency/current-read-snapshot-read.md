---
title: 当前读与快照读
---

# 当前读与快照读

在 MySQL InnoDB 引擎中，为了平衡并发读写性能，存在两种读操作模式：**快照读（Snapshot Read）** 和 **当前读（Current Read）**。它们的核心区别在于是否读取数据的最新版本，以及是否会加锁。

**快照读（Snapshot Read）**

快照读是指**读取数据的历史版本（快照）**，而非数据库当前最新版本。基于 MVCC（见 [MVCC & ReadView](mvcc-readview.md)）实现，**不加锁**，因此不会阻塞其他事务的读写操作，是提高并发性能的关键。

**实现原理**

1. **数据多版本**：InnoDB 通过 `undo log` 保存数据的历史版本（修改前的状态）
2. **Read View（读视图）**：事务启动时生成一个 Read View，记录当前活跃的事务 ID，用于判断哪些版本的数据对当前事务可见
    - 在 `Read Committed` 隔离级别下：每次 `SELECT` 都会生成新的 Read View，因此只能看到已提交的最新版本
    - 在 `Repeatable Read` 隔离级别下：整个事务只生成一次 Read View，因此多次查询能看到相同的快照（实现 "可重复读"）

**特点**

- 不加锁，读写不冲突，并发性能高
- 可能读取的不是最新数据（是历史快照）

**当前读（Current Read）**

当前读是指**读取数据的最新版本**，并且会对读取的记录加锁，阻塞其他事务的写操作（避免脏写）。

```sql
-- 加锁的SELECT语句
SELECT * FROM table WHERE ... FOR UPDATE;  -- 加排他锁（X锁）
SELECT * FROM table WHERE ... LOCK IN SHARE MODE;  -- 加共享锁（S锁）

-- 写操作（隐含当前读，需先读取最新数据再修改）
INSERT、UPDATE、DELETE
```

### 实现原理

1. 读取数据时，直接获取当前最新版本，而非历史快照
2. 对读取的记录加锁（具体锁类型取决于操作）：
    - `FOR UPDATE` 加排他锁（X 锁）：其他事务不能加 S 锁或 X 锁
    - `LOCK IN SHARE MODE` 加共享锁（S 锁）：其他事务可以加 S 锁，但不能加 X 锁
    - 写操作（`UPDATE` 等）会自动加排他锁，确保修改的是最新数据，且不会被其他事务干扰

> 当前读，快照读和 MVCC 之间是什么关系呢？
>
> - MVCC 多版本并发控制只是一个概念，并非具体实现
> - MySQL 实现 MVCC 概念中，快照读就是 MySQL 为我们实现 MVCC 理想模型的其中一个具体非阻塞读功能。而相对而言，当前读就是悲观锁的具体功能实现
> - MVCC 模型在 MySQL 中的具体实现则是由三个隐式字段、undo 日志、Read View 等去完成的
