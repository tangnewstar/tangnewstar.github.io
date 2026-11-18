---
title: 示例：RC 只锁修改的行
---

# 示例：RC 只锁修改的行

> 本文是 [事务隔离级别](transaction-isolation.md) 的配套示例。

Consider the table created and populated like this:

```sql
CREATE TABLE t (a INT NOT NULL, b INT) ENGINE = InnoDB;
INSERT INTO t VALUES (1,2),(2,3),(3,2),(4,3),(5,2);
COMMIT;
```

In this case, the table has no indexes, so searches and index scans use the hidden clustered index for record locking rather than indexed columns. 没有索引列，扫描全表（隐藏的 clustered index）。

Suppose that one session performs an `UPDATE`, and a second session performs an `UPDATE` by executing these statements following those of the first session:

```sql
# 数据：(1,2),(2,3),(3,2),(4,3),(5,2)

# Session A
START TRANSACTION;
UPDATE t SET b = 5 WHERE b = 3;

# Session B
UPDATE t SET b = 4 WHERE b = 2;

# >>> REPEATABLE READ
# Session A
x-lock(1,2); retain x-lock
x-lock(2,3); update(2,3) to (2,5); retain x-lock
x-lock(3,2); retain x-lock
x-lock(4,3); update(4,3) to (4,5); retain x-lock
x-lock(5,2); retain x-lock

# Session B
x-lock(1,2); block and wait for first UPDATE to commit or roll back

# >>> Read Committed
# Session A
x-lock(1,2); update(1,2) to (1,4); retain x-lock
x-lock(2,3); unlock(2,3)
x-lock(3,2); update(3,2) to (3,4); retain x-lock
x-lock(4,3); unlock(4,3)
x-lock(5,2); update(5,2) to (5,4); retain x-lock
```

As `InnoDB` executes each `UPDATE`, it first acquires an exclusive lock for each row, and then determines whether to modify it. If `InnoDB` does not modify the row, it releases the lock. Otherwise, `InnoDB` retains the lock until the end of the transaction.

When using the default `REPEATABLE READ` isolation level:

- the first `UPDATE` acquires an x-lock on each row that it reads and does not release any of them
- The second `UPDATE` blocks as soon as it tries to acquire any locks (because first update has retained locks on all rows), and does not proceed until the first `UPDATE` commits or rolls back

If `READ COMMITTED` is used instead:

- the first `UPDATE` acquires an x-lock on each row that it reads and releases those for rows that it does not modify
- For the second `UPDATE`, `InnoDB` does a "semi-consistent" read, returning the latest committed version of each row that it reads to MySQL so that MySQL can determine whether the row matches the `WHERE` condition of the `UPDATE`

However, if the `WHERE` condition includes an indexed column, and `InnoDB` uses the index, only the indexed column is considered when taking and retaining record locks.

In the following example, the first `UPDATE` takes and retains an x-lock on each row where b = 2. The second `UPDATE` blocks when it tries to acquire x-locks on the same records, as it also uses the index defined on column b.

```sql
CREATE TABLE t (a INT NOT NULL, b INT, c INT, INDEX (b)) ENGINE = InnoDB;
INSERT INTO t VALUES (1,2,3),(2,2,4);
COMMIT;

# Session A
START TRANSACTION;
UPDATE t SET b = 3 WHERE b = 2 AND c = 3;

# Session B
UPDATE t SET b = 4 WHERE b = 2 AND c = 4;
```
