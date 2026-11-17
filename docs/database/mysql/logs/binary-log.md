---
title: Binary Log
---

# Binary Log

**binlog 记录哪些内容？**

- 数据表的创建和数据修改

    The binary log contains "events" that describe database changes such as table creation operations or changes to table data.

- 更新数据时每个语句执行的时间

    The binary log also contains information about how long each statement took that updated data.

- 基于 Statement 的模式下，即使一条 SQL 没有修改数据也可能记录

    It also contains events for statements that potentially could have made changes (for example, a `DELETE` which matched no rows), unless row-based logging is used.

The binary log has two important purposes:

- **For replication**, the binary log on a replication source server provides a record of the data changes to be sent to replicas.

    > The source sends the information contained in its binary log to its replicas, which reproduce those transactions to make the same data changes that were made on the source.

- **Certain data recovery** operations require use of the binary log.

    > After a backup has been restored, the events in the binary log that were recorded after the backup was made are re-executed. These events bring databases up to date from the point of the backup.

### 事务原子性

The binary log is resilient to unexpected halts. Only complete events or transactions are logged or read back.

!!! note "提示"
    binlog的设计能够承受数据库的意外停止，并且保证只**记录或回放完整的事件或事务**，从而避免数据损坏或逻辑错误。

### 写入时机

Binary logging is done immediately after a statement or transaction completes but before any locks are released or any commit is done. This ensures that the log is logged in commit order.

Updates to non-transactional tables are stored in the binary log immediately after execution.

Within an uncommitted transaction, all updates (`UPDATE`, `DELETE`, or `INSERT`) that change transactional tables such as `InnoDB` tables are cached until a `COMMIT` statement is received by the server. At that point, [mysqld](https://dev.mysql.com/doc/refman/8.4/en/mysqld.html) writes the entire transaction to the binary log before the `COMMIT` is executed.

Modifications to nontransactional tables cannot be rolled back. If a transaction that is rolled back includes modifications to nontransactional tables, the entire transaction is logged with a `ROLLBACK` statement at the end to ensure that the modifications to those tables are replicated.

!!! note "提示"
    binlog记录完成在语句或事务完成之后，在锁释放和任何提交之前。这确保了日志记录是按事务提交的顺序，如果是非事务表binlog在执行后立即记录。

    在未提交事务中，所有对事务表的修改都会缓存直到COMMIT语句被server接收。在哪个时间点，mysqld将整个事务写入到binlog。（**先写binlog，再COMMIT**）

    对非事务型表（MyISAM）的修改无法回滚。如果一个被回滚的事务包含了对非事务型表的修改，整个事务会被记录到日志中，并在末尾加上一条 ROLLBACK 语句，以确保对那些非事务型表的修改也能被正确地复制。

### 写入过程

When a thread that handles the transaction starts, it allocates a buffer of `binlog_cache_size` to buffer statements. If a statement is bigger than this, the thread opens a temporary file to store the transaction. The temporary file is deleted when the thread ends. If binary log encryption is active on the server, the temporary file is encrypted.

!!! note "提示"
    线程为 binlog 事件分配**内存缓冲区**，不足时使用**磁盘临时文件**。

If you are using the binary log and row based logging, concurrent inserts are converted to normal inserts for `CREATE ... SELECT` or `INSERT ... SELECT` statements. This is done to ensure that you can re-create an exact copy of your tables by applying the log during a backup operation. If you are using statement-based logging, the original statement is written to the log.

!!! note "提示"
    如果你使用基于行的日志，并发插入会被转为普通插入。这样做是为了确保在备份操作应用日志时，可以重新创建表的精确副本。

    如果使用的是基于语句的日志，原始语句会被写入日志

### 同步时机

By default, the binary log is synchronized to disk at each write (`sync_binlog=1`). If `sync_binlog` was not enabled, and the operating system or machine (not only the MySQL server) crashed, there is a chance that the last statements of the binary log could be lost. To prevent this, enable the `sync_binlog` system variable to synchronize the binary log to disk after every **N** commit groups.

> The safest value for `sync_binlog` is 1 (the default), but this is also the slowest.

### 2PC

In earlier MySQL releases, there was a chance of inconsistency between the table content and binary log content if a crash occurred, even with [`sync_binlog`](https://dev.mysql.com/doc/refman/8.4/en/replication-options-binary-log.html#sysvar_sync_binlog) set to 1. For example, if you are using `InnoDB` tables and the MySQL server processes a [`COMMIT`](https://dev.mysql.com/doc/refman/8.4/en/commit.html) statement, it writes many prepared transactions to the binary log in sequence, synchronizes the binary log, and then commits the transaction into `InnoDB`. If the server unexpectedly exited between those two operations, the transaction would be rolled back by `InnoDB` at restart but still exist in the binary log. Such an issue was resolved in previous releases by enabling `InnoDB` support for two-phase commit in XA transactions. In MySQL 8.4, `InnoDB` support for two-phase commit in XA transactions is always enabled.

!!! note "提示"
    在早期的MySql版本中，在服务崩溃后即使`sync_binlog=1`，存储引擎和binlog数据有可能不一致。

    比如说使用InnoDB表，Server在处理一个COMMIT语句，按顺序写入很多准备好的事务到binlog中，同步binlog然后提交事务到InnoDB。如果server未预料在两个步骤（同步binglog之后，提交到InnoDB前）退出了，事务会在重启后被InnoDB回滚但仍然存在于binlog中。

    这样的问题在之前的版本**支持启用InnoDB对XA事务的两阶段提交**，在8.4默认启用

    *InnoDB Prepare → binlog → InnoDB Commit*，重启后扫描 InnoDB Prepare状态

    如果binlog存在，在InnoDB重新提交（前滚）如果不存在则在InnoDB回滚

`InnoDB` support for two-phase commit in XA transactions ensures that the binary log and `InnoDB` data files are synchronized. However, the MySQL server should also be configured to synchronize the binary log and the `InnoDB` logs to disk before committing the transaction. The `InnoDB` logs are synchronized by default, and `sync_binlog=1` ensures the binary log is synchronized. The effect of implicit `InnoDB` support for two-phase commit in XA transactions and `sync_binlog=1` is that at restart after a crash, after doing a rollback of transactions, the MySQL server scans the latest binary log file to collect transaction **xid** values and calculate the last valid position in the binary log file. The MySQL server then tells `InnoDB` to complete any prepared transactions that were successfully written to the to the binary log, and truncates the binary log to the last valid position. This ensures that the binary log reflects the exact data of `InnoDB` tables, and therefore the replica remains in synchrony with the source because it does not receive a statement which has been rolled back.

!!! note "提示"
    InnoDB 对 XA 两阶段提交的支持，从**逻辑上**为 binlog 和 InnoDB 数据文件提供了同步的框架。它建立了事务在两者之间"同生共死"的协调规则。
    仅有逻辑框架不够，还必须从**物理上**保证日志确实写入了磁盘，而不仅仅是缓存在内存中。这是通过配置实现的：`sync_binlog=1`：确保每次事务提交时，binlog 都强制刷盘 (**fsync**)。
    MySQL 重启后，首先会进行一个初步的恢复和回滚。然后，它会**扫描最新的 binlog 文件**。
    收集所有在 binlog 中找到的事务的 **xid** (XA 事务的唯一标识符)。计算出 binlog 中**最后一个完整有效的事件的位置**。
    MySQL 服务器将扫描 binlog 得到的 **xid** 列表与 InnoDB 中处于 **PREPARE** 状态的事务进行比对。

    - **如果**一个事务在 InnoDB 中是 **PREPARE** 状态，并且它的 **xid** 存在于 binlog 中 → 这说明该事务在崩溃前已经成功完成了两阶段提交的第二步（写binlog）。服务器会命令 InnoDB **提交**这个事务。
    - **如果** binlog 的尾部因为崩溃而损坏（即"短了"），MySQL 会主动将 binlog 文件**截断**到之前计算出的"最后一个有效位置"。

    从库在重放这份 binlog 时，**永远不会接收到一个实际上在主库已被回滚了的语句**。从而保证了主从数据的严格同步。

### 故障处理

If the MySQL server discovers at crash recovery that the binary log is shorter than it should have been, it lacks at least one successfully committed `InnoDB` transaction. This should not happen if `sync_binlog=1` and the disk/file system do an actual sync when they are requested to (some do not), so the server prints an error message `The binary log file_name is shorter than its expected size`. In this case, this binary log is not correct and replication should be restarted from a fresh snapshot of the source's data.

!!! note "提示"
    如果在故障恢复时server发现binlog比实际短（例如缺少在InnoDB成功提交的事务）。

    这在sync_binlog=1下不应该发生，磁盘和文件系统被fsync请求时有时不会执行真的同步。

    此时server打印`The binary log file_name is shorter than its expected size` binlog不正确需要重启副本，使用源数据的全新快照数据

If the server is unable to write to the binary log, flush binary log files, or synchronize the binary log to disk, the binary log on the replication source server can become inconsistent and replicas can lose synchronization with the source. The `binlog_error_action` system variable controls the action taken if an error of this type is encountered with the binary log.

- The default setting, `ABORT_SERVER`, makes the server halt binary logging and shut down. At this point, you can identify and correct the cause of the error. On restart, recovery proceeds as in the case of an unexpected server halt
- The setting `IGNORE_ERROR` provides backward compatibility with older versions of MySQL. With this setting, the server continues the ongoing transaction and logs the error, then halts binary logging, but continues to perform updates. At this point, you can identify and correct the cause of the error. To resume binary logging, `log_bin` must be enabled again, which requires a server restart. Only use this option if you require backward compatibility, and the binary log is non-essential on this MySQL server instance. For example, you might use the binary log only for intermittent auditing or debugging of the server, and not use it for replication from the server or rely on it for point-in-time restore operations.

### 性能

Running a server with binary logging enabled makes performance slightly slower. However, the benefits of the binary log in enabling you to set up replication and for restore operations generally outweigh this minor performance decrement.

!!! note "提示"
    binlog开启会牺牲性能，但是复制和备份能力比这点性能损失更重要

### 安全性

Passwords in statements written to the binary log are rewritten by the server not to occur literally in plain text.

MySQL binary log files and relay log files can be encrypted, helping to protect these files and the potentially sensitive data contained in them from being misused by outside attackers, and also from unauthorized viewing by users of the operating system where they are stored. You enable encryption on a MySQL server by setting the `binlog_encryption` system variable to `ON`.

!!! note "提示"
    为了保护安全，当 MySQL 服务器将某些包含密码的 SQL 语句写入二进制日志（binlog）时，它会自动将这些密码重写，避免其以明文形式出现。
