---
title: Consistent Read
---

# Consistent Read

A read operation that uses ***snapshot*** information to present query results based on a point in time, regardless of changes performed by other transactions running at the same time. If queried data has been changed by another transaction, the original data is reconstructed based on the contents of the ***undo log***. This technique avoids some of the ***locking*** issues that can reduce ***concurrency*** by forcing transactions to wait for other transactions to finish.

With ***REPEATABLE READ*** ***isolation level***, the snapshot is based on the time when the first read operation is performed. With ***READ COMMITTED*** isolation level, the snapshot is reset to the time of each consistent read operation.

Consistent read is the default mode in which `InnoDB` processes `SELECT` statements in ***READ COMMITTED*** and ***REPEATABLE READ*** isolation levels. Because a consistent read does not set any locks on the tables it accesses, other sessions are free to modify those tables while a consistent read is being performed on the table.
