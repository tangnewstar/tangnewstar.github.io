---
title: InnoDB Multi-Versioning
---

# InnoDB Multi-Versioning

`InnoDB` is a multi-version storage engine. It keeps information about old versions of changed rows to support transactional features such as concurrency and rollback. This information is stored in undo tablespaces in a data structure called a rollback segment.

`InnoDB` uses the information in the rollback segment to perform the undo operations needed in a transaction rollback. It also uses the information to build earlier versions of a row for a consistent read. [consistent read](https://app.notion.com/p/consistent-read-6428623ca9758338a6ff81198807d3e4?pvs=21)

!!! note "提示"
    InnoDB是多版本存储引擎。保有修改行的旧版本信息，以支持事务特性例如并发和回滚。

    旧版本的信息存储在 undo tablespace 中的一个叫做rollback segment的地方。

    - InnoDB使用rollback segment中的信息，当需要事务回滚时来执行回滚操作
    - 同时也用来构建一致性读中行数据的更早版本

Internally, `InnoDB` adds three fields to each row stored in the database:

• A 6-byte `DB_TRX_ID` field indicates the transaction identifier for the last transaction that inserted or updated the row. Also, a deletion is treated internally as an update where a special bit in the row is set to mark it as deleted.

• A 7-byte `DB_ROLL_PTR` field called the roll pointer. The roll pointer points to an undo log record written to the rollback segment. If the row was updated, the undo log record contains the information necessary to rebuild the content of the row before it was updated.

• A 6-byte `DB_ROW_ID` field contains a row ID that increases monotonically as new rows are inserted. If `InnoDB` generates a clustered index automatically, the index contains row ID values. Otherwise, the `DB_ROW_ID` column does not appear in any index.

!!! note "提示"
    InnoDB在每行数据上加了三个字段：

    - `DB_TRX_ID` ：上一次插入或更新行的事务标识
    - `DB_ROLL_PTR` ：回滚指针，指向写到rollback segment的undolog记录
    - `DB_ROW_ID` ：单调增长的行ID

Undo logs in the rollback segment are divided into insert and update undo logs. Insert undo logs are needed only in transaction rollback and can be discarded as soon as the transaction commits. Update undo logs are used also in consistent reads, but they can be discarded only after there is no transaction present for which `InnoDB` has assigned a snapshot that in a consistent read could require the information in the update undo log to build an earlier version of a database row.

!!! note "提示"
    Undolog在rollback segment中被分为插入undolog和更新undolog。

    插入undolog只在事务回滚需要，事务一旦提交就可以被丢弃

    更新undolog也用在一致性读，只有不存在InnoDB已经分配了快照（一致性读能够从更新undolog获取信息构建行数据的早期版本）的事务才能被丢弃

It is recommend that you commit transactions regularly, including transactions that issue only consistent reads. Otherwise, `InnoDB` cannot discard data from the update undo logs, and the rollback segment may grow too big, filling up the undo tablespace in which it resides.

!!! note "提示"
    推荐定期提交事务，包括那些只有一致性读的事务。why？

    要不然不能丢弃那些更新undolog，rollback segment会很大，填满undolog tablespace

The physical size of an undo log record in the rollback segment is typically smaller than the corresponding inserted or updated row. You can use this information to calculate the space needed for your rollback segment.

In the `InnoDB` multi-versioning scheme, a row is not physically removed from the database immediately when you delete it with an SQL statement. `InnoDB` only physically removes the corresponding row and its index records when it discards the update undo log record written for the deletion. This removal operation is called a purge, and it is quite fast, usually taking the same order of time as the SQL statement that did the deletion.

!!! note "提示"
    删除操作不立即移除物理数据，而是延迟通过purge 操作（为删除写的更新undolog丢弃后移除相应的行和索引记录）完成，purge操作非常快，通常和删除的SQL同一数量级

If you insert and delete rows in smallish batches at about the same rate in the table, the purge thread can start to lag behind and the table can grow bigger and bigger because of all the "dead" rows, making everything disk-bound and very slow. In such cases, throttle new row operations, and allocate more resources to the purge thread by tuning the `innodb_max_purge_lag` system variable.

!!! note "提示"
    当以相同的速率小批量的插入和删除时，purge线程可能会落后，表的体积越来越大（由于未物理删除的dead rows），导致所有操作受限于磁盘IO的很慢。

    这种情况下，可以限制新行操作和调优`innodb_max_purge_lag` 给purge线程分配更多资源

### 多版本和二级索引

`InnoDB` multiversion concurrency control (MVCC) treats secondary indexes differently than clustered indexes. Records in a clustered index are updated in-place, and their hidden system columns point undo log entries from which earlier versions of records can be reconstructed. Unlike clustered index records, secondary index records do not contain hidden system columns nor are they updated in-place.

!!! note "提示"
    InnoDB的MVCC对待二级索引和聚簇索引不同。

    聚簇索引就地更新，隐藏列（DB_TRX_ID，DB_ROLL_PTR) 指向undolog入口，以此重新构造早期版本的记录。

    不想聚簇索引，二级索引不包含隐藏列，不是就地更新的

When a secondary index column is updated, old secondary index records are delete-marked, new records are inserted, and delete-marked records are eventually purged. When a secondary index record is delete-marked or the secondary index page is updated by a newer transaction, `InnoDB` looks up the database record in the clustered index. In the clustered index, the record's `DB_TRX_ID` is checked, and the correct version of the record is retrieved from the undo log if the record was modified after the reading transaction was initiated.

!!! note "提示"
    当二级索引列被更新时，旧二级索引记录会被标记为"删除"并最终被purge，新的记录被插入。

    当二级索引记录是"删除"标记或者二级索引页被新的事务更新时，

    - 查询二级索引的聚簇索引记录
    - 检查DB_TRX_ID，如果记录修改在读取的事务初始化后，从undolog取出正确版本

If a secondary index record is marked for deletion or the secondary index page is updated by a newer transaction, the [covering index](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_covering_index) technique is not used. Instead of returning values from the index structure, `InnoDB` looks up the record in the clustered index.

!!! note "提示"
    如果二级索引记录是"删除"标记或者二级索引页被新的事务更新时，覆盖索引失效

    而是返回索引结构的值，InnoDB查找聚簇索引的记录 **为什么必须查询聚簇索引？**

However, if the [index condition pushdown (ICP)](https://dev.mysql.com/doc/refman/8.0/en/index-condition-pushdown-optimization.html) optimization is enabled, and parts of the `WHERE` condition can be evaluated using only fields from the index, the MySQL server still pushes this part of the `WHERE` condition down to the storage engine where it is evaluated using the index. If no matching records are found, the clustered index lookup is avoided. If matching records are found, even among delete-marked records, `InnoDB` looks up the record in the clustered index.

!!! note "提示"
    ICP 是 MySQL 的一种性能优化技术，它允许将 **部分 WHERE 条件 "下推" 到存储引擎层（而非在 MySQL 服务器层）**，直接利用二级索引中的字段进行过滤。

    但是如果ICP优化启用，where条件的部分条件可以仅通过 二级索引中的字段计算，server会将这部分条件下推到存储引擎，因此使用二级索引过滤。

    - 如果没有匹配记录，避免聚簇索引查询
    - 有匹配记录，必须查询聚簇索引（即便包含标记删除的数据）
