---
title: 锁机制
---

# 锁机制

锁机制是 InnoDB 实现事务隔离性的底层手段。官方说明：锁请求如果与现有锁兼容则可授予；如果冲突，请求事务会等待，直到冲突锁释放；如果冲突请求可能导致死锁，则会发生错误。

按照锁定粒度，可分为**表锁**和**行锁**；InnoDB 在行锁之上又实现了共享锁、排他锁、意向锁、记录锁、间隙锁、临键锁、插入意向锁与 AUTO-INC 锁等多种类型。

## 表锁与行锁

- **表级锁（Table Lock）**：锁定整张表，由 MySQL 服务层或 InnoDB 的意向锁实现。开销小、加锁快，但并发粒度粗。
- **行级锁（Row Lock）**：InnoDB 支持，只锁定被访问的索引记录。并发粒度细，但开销更大，且可能出现死锁。

!!! note "行锁总是加在索引上"
    官方指出：记录锁总是锁定**索引记录**。即使表没有显式索引，InnoDB 也会创建隐藏聚簇索引来实现行锁。因此行锁实际依赖索引，这也是“索引失效可能导致锁表”的根源。

## InnoDB 锁类型

### 共享锁与排他锁

- **共享锁（S Lock）**：允许持有锁的事务**读取**一行。
- **排他锁（X Lock）**：允许持有锁的事务**更新或删除**一行。

冲突规则：

| 当前锁 \ 请求锁 | S | X |
|---|---|---|
| **S** | 兼容 | 冲突 |
| **X** | 冲突 | 冲突 |

### 意向锁

意向锁是**表级锁**，用于表明事务稍后打算对表中的某些行加什么类型的锁：

- **IS（意向共享锁）**：表示事务打算对表中的行设置共享锁，如 `SELECT ... FOR SHARE`。
- **IX（意向排他锁）**：表示事务打算对表中的行设置排他锁，如 `SELECT ... FOR UPDATE`。

加锁协议：事务要获得某行的 S 锁，必须先获得表上的 `IS` 或更强锁；要获得某行的 X 锁，必须先获得表上的 `IX` 锁。意向锁不会阻塞除全表请求（如 `LOCK TABLES ... WRITE`）以外的操作。

表级锁兼容矩阵：

| 锁模式 | X | IX | S | IS |
|---|---|---|---|---|
| **X** | 冲突 | 冲突 | 冲突 | 冲突 |
| **IX** | 冲突 | 兼容 | 冲突 | 兼容 |
| **S** | 冲突 | 冲突 | 兼容 | 兼容 |
| **IS** | 冲突 | 兼容 | 兼容 | 兼容 |

### 记录锁 Record Lock

记录锁是**索引记录上的锁**。例如：

```sql
SELECT c1 FROM t WHERE c1 = 10 FOR UPDATE;
```

会阻止其他事务插入、更新或删除 `t.c1 = 10` 的行。记录锁遵循 S/X 冲突规则。

### 间隙锁 Gap Lock

间隙锁锁定**索引记录之间的间隙**，或第一条记录之前、最后一条记录之后的间隙。例如：

```sql
SELECT c1 FROM t WHERE c1 BETWEEN 10 AND 20 FOR UPDATE;
```

会阻止其他事务向该间隙插入 `15` 这样的值，即使当前并不存在该值。

官方要点：

- 间隙锁是“纯粹抑制性”的，唯一目的就是阻止其他事务向间隙插入记录。
- 间隙锁可以共存：一个事务持有的间隙锁不会阻止另一个事务在同一间隙加锁，S 间隙锁与 X 间隙锁没有区别、彼此不冲突。
- 使用唯一索引搜索唯一行时**不需要**间隙锁；但如果搜索条件只覆盖多列唯一索引的一部分，仍可能发生间隙锁定。
- 在 `READ COMMITTED` 隔离级别下，搜索和索引扫描的间隙锁定被禁用，仅用于外键约束检查和重复键检查。

### 临键锁 Next-Key Lock

临键锁是**记录锁与记录之前间隙锁的组合**：既锁住索引记录本身，又锁住该记录前面的间隙。

- 如果一个会话对索引记录 `R` 持有 S 锁或 X 锁，其他会话不能在 `R` 之前的紧邻间隙中插入新记录。
- 在默认的 `REPEATABLE READ` 隔离级别下，InnoDB 对搜索和索引扫描使用临键锁，以防止幻读。
- 官方示例：索引值为 `10, 11, 13, 20` 时，可能的临键锁区间为：

```text
(负无穷, 10]
(10, 11]
(11, 13]
(13, 20]
(20, 正无穷)
```

其中最后的开区间锁住的是最大索引值之后的间隙（对应 supremum 伪记录）。

### 插入意向锁与 AUTO-INC 锁

- **插入意向锁（Insert Intention Lock）**：`INSERT` 操作在插入行之前设置的一种间隙锁，用于表明插入意图。多个事务在同一间隙插入**不同位置**时不必互相等待；但若其他事务已在该间隙持有间隙锁或临键锁，插入意向锁需要等待。
- **AUTO-INC 锁**：事务向包含 `AUTO_INCREMENT` 列的表插入数据时获取的一种特殊**表级锁**。最简单情况下，一个事务插入时其他事务必须等待，以便自增值连续。其具体行为由 `innodb_autoinc_lock_mode` 控制，用于权衡自增值的可预测性与插入并发性。

## 死锁

### 定义

官方定义：

> A deadlock is a situation in which multiple transactions are unable to proceed because each transaction holds a lock that is needed by another one. Because all transactions involved are waiting for the same resource to become available, none of them ever releases the lock it holds.

即：多个事务都无法继续执行，因为每个事务都持有另一个事务需要的锁；所有相关事务都在等待资源释放，因此没有任何事务会主动释放自己持有的锁。

常见场景：

1. 多个事务以**相反顺序**锁定多张表中的行（如通过 `UPDATE` 或 `SELECT ... FOR UPDATE`）。
2. 多个事务锁定索引记录和间隙时，因执行时序各自获得部分锁而无法继续。

### 死锁检测与处理

- **死锁检测**：`innodb_deadlock_detect` 默认启用。发生死锁时，InnoDB 检测到该情况并选择一个事务作为 **victim（牺牲者）** 回滚，其他事务得以继续。
- **锁等待超时**：如果禁用死锁检测（`innodb_deadlock_detect=OFF`），InnoDB 依赖 `innodb_lock_wait_timeout` 处理锁等待超时。
- 官方强调：即使应用逻辑正确，应用仍必须处理**事务被回滚、需要重试**的情况。

### 如何减少死锁

官方建议包括：

1. 使用事务而不是 `LOCK TABLES` 语句。
2. 保持事务短小，避免长时间打开。
3. 不同事务更新多张表或大范围行时，使用**相同的操作顺序**。
4. 为 `SELECT ... FOR UPDATE` 和 `UPDATE ... WHERE` 使用的列建立索引，减少不必要的锁范围。
5. 注意范围锁与间隙锁，死锁可能因执行时序造成部分锁已获取、部分未获取而发生。
6. 不要指望通过调整隔离级别消除死锁——隔离级别主要改变读操作行为，而死锁来自写操作。
7. 应用必须实现重试处理。

诊断手段：

- `SHOW ENGINE INNODB STATUS` 查看最近一次 InnoDB 死锁信息。
- 频繁死锁时启用 `innodb_print_all_deadlocks`，将死锁信息打印到错误日志。

## 相关文档

- [事务隔离级别](transaction-isolation.md)
- [并发事务问题](multi-transaction-concurrency.md)
- [MVCC & ReadView](mvcc-readview.md)

## 参考

- [MySQL 8.0 Reference Manual · 17.7.1 InnoDB Locking](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)
- [MySQL 8.0 Reference Manual · 17.7.5 Deadlocks in InnoDB](https://dev.mysql.com/doc/refman/8.0/en/innodb-deadlocks.html)
