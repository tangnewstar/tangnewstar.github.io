---
title: Undo Log
---

# Undo Log

An undo log is a collection of undo log records associated with a single read-write transaction. An undo log record contains information about how to undo the latest change by a transaction to a [clustered index](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_clustered_index) record.

If another transaction needs to see the original data as part of a consistent read operation, the unmodified data is retrieved from undo log records.

> *see also [consistent read](https://app.notion.com/p/consistent-read-6428623ca9758338a6ff81198807d3e4?pvs=21)*

Undo logs exist within [undo log segments](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_undo_log_segment), which are contained within [rollback segments](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_rollback_segment). Rollback segments reside in [undo tablespaces](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_undo_tablespace) and in the [global temporary tablespace](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_global_temporary_tablespace).

Undo logs that reside in the global temporary tablespace are used for transactions that modify data in user-defined temporary tables. These undo logs are not redo-logged, as they are not required for crash recovery. They are used only for rollback while the server is running. This type of undo log benefits performance by avoiding redo logging I/O.

!!! note "提示"
    **驻留在全局临时表空间中的 undo logs 用于修改用户定义的临时表中数据的事务。这些 undo logs 不会被重做日志（redo log）记录，因为它们对于崩溃恢复来说不是必需的。它们仅用于在服务器运行时进行回滚操作。这种类型的 undo log 通过避免重做日志的 I/O 操作来提升性能。**

Each undo tablespace and the global temporary tablespace individually support a maximum of 128 rollback segments. The `innodb_rollback_segments` variable defines the number of rollback segments.

A transaction is assigned up to four undo logs, one for each of the following operation types:

1. [`INSERT`](https://dev.mysql.com/doc/refman/8.0/en/insert.html) operations on user-defined tables
2. [`UPDATE`](https://dev.mysql.com/doc/refman/8.0/en/update.html) and [`DELETE`](https://dev.mysql.com/doc/refman/8.0/en/delete.html) operations on user-defined tables
3. [`INSERT`](https://dev.mysql.com/doc/refman/8.0/en/insert.html) operations on user-defined temporary tables
4. [`UPDATE`](https://dev.mysql.com/doc/refman/8.0/en/update.html) and [`DELETE`](https://dev.mysql.com/doc/refman/8.0/en/delete.html) operations on user-defined temporary tables
