# 日志

MySQL 的日志分为 Server 层和存储引擎层，这里按所属层级整理。

### 总览与详解

- [MySQL 日志详解](mysql-logs-overview.md) — 日志类型总览、三种日志原理与协作、SQL 执行过程与崩溃恢复

### Server 层日志

- [Binary Log（binlog）](binary-log.md)

### 存储引擎日志（InnoDB）

- [Redo Log（重做日志）](redo-log.md)
- [Undo Log（回滚日志）](undo-log.md)
