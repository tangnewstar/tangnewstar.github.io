---
title: LSN
---

# LSN

Acronym for "log sequence number".

This arbitrary, ever-increasing value represents a point in time corresponding to operations recorded in the ***redo log***. (This point in time is regardless of ***transaction*** boundaries; it can fall in the middle of one or more transactions.)

It is used internally by `InnoDB` during ***crash recovery*** and for managing the ***buffer pool***.

> Prior to MySQL 5.6.3, the LSN was a 4-byte unsigned integer. The LSN became an 8-byte unsigned integer in MySQL 5.6.3 when the redo log file size limit increased from 4GB to 512GB, as additional bytes were required to store extra size information. Applications built on MySQL 5.6.3 or later that use LSN values should use 64-bit rather than 32-bit variables to store and compare LSN values.

> In the ***MySQL Enterprise Backup*** product, you can specify an LSN to represent the point in time from which to take an ***incremental backup***. The relevant LSN is displayed by the output of the **mysqlbackup** command. Once you have the LSN corresponding to the time of a full backup, you can specify that value to take a subsequent incremental backup, whose output contains another LSN for the next incremental backup.

LSN 是 "log sequence number"（日志序列号）的缩写。

这是一个任意且不断递增的值，表示对应于记录在 ***重做日志*** 中操作的某个时间点（这个时间点与 ***事务*** 边界无关，可以落在一个或多个事务的中间）。

LSN 在 `InnoDB` 内部用于 ***崩溃恢复*** 以及管理 ***缓冲池***。

> 在 MySQL 5.6.3 之前，LSN 是一个 4 字节无符号整数。MySQL 5.6.3 起，随着重做日志文件大小限制从 4GB 增加到 512GB，LSN 变为 8 字节无符号整数，以存储额外的大小信息。基于 MySQL 5.6.3 或更高版本的应用，如使用 LSN 值，应使用 64 位变量来存储和比较 LSN，而非 32 位。

> 在 ***MySQL Enterprise Backup*** 产品中，可以通过指定 LSN 来表示增量备份的起始时间点。相关的 LSN 会在 **mysqlbackup** 命令的输出中显示。获取全量备份时对应的 LSN 后，可以指定该值进行下次增量备份，而每个增量备份的输出中会包含下一个增量备份所需的 LSN。
