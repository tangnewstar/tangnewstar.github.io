---
title: Redo Log
---

# Redo Log

The redo log is a disk-based data structure used during crash recovery to correct data written by incomplete transactions.

During normal operations, the redo log encodes requests to change table data that result from SQL statements or low-level API calls.

Modifications that did not finish updating data files before an unexpected shutdown are replayed automatically during initialization and before connections are accepted.

The redo log is physically represented on disk by redo log files. Data that is written to redo log files is encoded in terms of records affected, and this data is collectively referred to as redo. The passage of data through redo log files is represented by an ever-increasing [LSN](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_lsn) value. Redo log data is appended as data modifications occur, and the oldest data is truncated as the checkpoint progresses.

!!! note "提示"
    redo log 是由物理磁盘的redo log文件表示

    写入redo log文件的数据是根据受影响的行数编码的，这些数据统称为redo

    数据在redolog文件中的通过情况，由一个不断增长的LSN值表示

    当数据修改发生时，redolog数据被追加写入，并随着checkpoint的推进最旧的数据被截断
