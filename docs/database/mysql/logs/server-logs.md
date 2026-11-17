---
title: Server Logs
---

# Server Logs

| **Log Type** | **Information Written to Log** |
| --- | --- |
| Error log | Problems encountered starting, running, or stopping [mysqld](https://dev.mysql.com/doc/refman/8.4/en/mysqld.html) |
| General query log | Established client connections and statements received from clients |
| **Binary log** | Statements that change data (also used for replication) |
| Relay log | Data changes received from a replication source server |
| Slow query log | Queries that took more than [`long_query_time`](https://dev.mysql.com/doc/refman/8.4/en/server-system-variables.html#sysvar_long_query_time) seconds to execute |
| DDL logs | Atomic DDL operations performed by DDL statements |
