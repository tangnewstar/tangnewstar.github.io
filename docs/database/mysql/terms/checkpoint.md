---
title: Checkpoint
---

# Checkpoint

As changes are made to data pages that are cached in the ***buffer pool***, those changes are written to the ***data files*** sometime later, a process known as ***flushing***. The checkpoint is a record of the latest changes (represented by an ***LSN*** value) that have been successfully written to the data files.

*fuzzy checkpointing*

A technique that ***flushes*** small batches of ***dirty pages*** from the ***buffer pool***, rather than flushing all dirty pages at once which would disrupt database processing.

*sharp checkpointing*

In contrast, sharp checkpointing writes all dirty pages from the buffer pool to the data files in a single batch, which can momentarily pause database activity but ensures that the buffer pool and data files are fully synchronized at the checkpoint.

**Fuzzy checkpoint: Used by InnoDB during normal operations to periodically flush some dirty pages.**

**Sharp checkpoint: Triggered during database shutdown to flush all dirty pages at once.**

### How Checkpoint Processing Works

`InnoDB` implements a [checkpoint](https://dev.mysql.com/doc/refman/8.4/en/glossary.html#glos_checkpoint) mechanism known as [fuzzy checkpointing](https://dev.mysql.com/doc/refman/8.4/en/glossary.html#glos_fuzzy_checkpointing). `InnoDB` flushes modified database pages from the buffer pool in small batches. There is no need to flush the buffer pool in one single batch, which would disrupt processing of user SQL statements during the checkpointing process.

During [crash recovery](https://dev.mysql.com/doc/refman/8.4/en/glossary.html#glos_crash_recovery), `InnoDB` looks for a checkpoint label written to the log files. It knows that all modifications to the database before the label are present in the disk image of the database. Then `InnoDB` scans the log files forward from the checkpoint, applying the logged modifications to the database.
