---
title: 事务隔离级别
---

# 事务隔离级别

Transaction isolation is one of the foundations of database processing. Isolation is the I in the acronym ACID; the isolation level is the setting that fine-tunes the balance between performance and reliability, consistency, and reproducibility of results when multiple transactions are making changes and performing queries at the same time.

`InnoDB` offers all four transaction isolation levels described by the SQL:1992 standard: `READ UNCOMMITTED`, `READ COMMITTED`, `REPEATABLE READ`, and `SERIALIZABLE`. The default isolation level for `InnoDB` is `REPEATABLE READ`.

`InnoDB` supports each of the transaction isolation levels described here using different locking strategies. You can enforce a high degree of consistency with the default `REPEATABLE READ` level, for operations on crucial data where ACID compliance is important. Or you can relax the consistency rules with `READ COMMITTED` or even `READ UNCOMMITTED`, in situations such as bulk reporting where precise consistency and repeatable results are less important than minimizing the amount of overhead for locking. `SERIALIZABLE` enforces even stricter rules than `REPEATABLE READ`, and is used mainly in specialized situations, such as with XA transactions and for troubleshooting issues with concurrency and deadlocks.

## REPEATABLE READ

InnoDB 默认的隔离级别。

Consistent reads within the same transaction read the snapshot established by the first read. This means that if you issue several plain (nonlocking) `SELECT` statements within the same transaction, these `SELECT` statements are consistent also with respect to each other.

For locking reads (`SELECT` with `FOR UPDATE` or `FOR SHARE`), `UPDATE`, and `DELETE` statements, locking depends on whether the statement uses a unique index with a unique search condition, or a range-type search condition.

- For a unique index with a unique search condition, `InnoDB` locks only the index record found, not the gap before it.
- For other search conditions, `InnoDB` locks the index range scanned, using gap locks or next-key locks to block insertions by other sessions into the gaps covered by the range.

## READ COMMITTED

Each consistent read, even within the same transaction, sets and reads its own fresh snapshot.

For locking reads (`SELECT` with `FOR UPDATE` or `FOR SHARE`), `UPDATE` statements, and `DELETE` statements, `InnoDB` locks only index records, not the gaps before them, and thus permits the free insertion of new records next to locked records. Gap locking is only used for foreign-key constraint checking and duplicate-key checking.

Because gap locking is disabled, phantom row problems may occur, as other sessions can insert new rows into the gaps.

Only row-based binary logging is supported with the `READ COMMITTED` isolation level. If you use `READ COMMITTED` with `binlog_format=MIXED`, the server automatically uses row-based logging.

Using `READ COMMITTED` has additional effects:

- For `UPDATE` or `DELETE` statements, `InnoDB` holds locks only for rows that it updates or deletes. Record locks for nonmatching rows are released after MySQL has evaluated the `WHERE` condition. This greatly reduces the probability of deadlocks, but they can still happen.
- For `UPDATE` statements, if a row is already locked, `InnoDB` performs a "semi-consistent" read, returning the latest committed version to MySQL so that MySQL can determine whether the row matches the `WHERE` condition of the `UPDATE`. If the row matches (must be updated), MySQL reads the row again and this time `InnoDB` either locks it or waits for a lock on it.

RC 只锁修改的行，具体见 [示例：RC 只锁修改的行](example-rc-locks.md)。

## READ UNCOMMITTED

`SELECT` statements are performed in a nonlocking fashion, but a possible earlier version of a row might be used. Thus, using this isolation level, such reads are not consistent. This is also called a dirty read.

## SERIALIZABLE

This level is like `REPEATABLE READ`, but `InnoDB` implicitly converts all plain `SELECT` statements to `SELECT ... FOR SHARE` if `autocommit` is disabled. If `autocommit` is enabled, the `SELECT` is its own transaction.

- autocommit：0，Serializable 级别会**隐式**将所有普通 select 转为 `select… for share`（共享锁 Shared Lock，意味着其他事务不能修改），**这个锁一直持有到事务结束**
- autocommit：1，Serializable 级别会将所有普通 select 加上共享锁，同时这个 select 是个独立的微型事务，**select 结束立即释放锁**

It therefore is known to be read only and can be serialized if performed as a consistent (nonlocking) read and need not block for other transactions.

因此如果一个事务确认为只读的，可以使用 Consistent (nonblocking) Read 来执行。读取历史快照，满足了 Serializable 级别要求，而且不需要锁定其他事务。**如果 InnoDB 能够确定一个事务是只读的（READ ONLY TRANSACTION），它就会为了提升性能而选择不使用默认的加锁方式，而是退回到使用和 REPEATABLE-READ 级别一样的"一致性非锁定读"（MVCC 快照）**。

## 配套示例

- [示例：RC 只锁修改的行](example-rc-locks.md)
- [示例：分析不同的隔离级别](example-isolation-levels.md)
