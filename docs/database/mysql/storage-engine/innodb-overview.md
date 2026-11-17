---
title: InnoDB Overview
---

# InnoDB Overview

`InnoDB` is a general-purpose storage engine that balances high reliability and high performance.

### Advantages

- Its DML operations follow the ACID model, with transactions featuring commit, rollback, and crash-recovery capabilities to protect user data.
- Row-level locking and Oracle-style consistent reads increase multi-user concurrency and performance.
- `InnoDB` tables arrange your data on disk to optimize queries based on primary keys. Each `InnoDB` table has a primary key index called the clustered index that organizes the data to minimize I/O for primary key lookups.
- To maintain data integrity, `InnoDB` supports `FOREIGN KEY` constraints. With foreign keys, inserts, updates, and deletes are checked to ensure they do not result in inconsistencies across related tables.

### Features

***B-tree indexes, Backup/point-in-time recovery***

### ACID Model

The [ACID](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_acid) model is a set of database design principles that emphasize aspects of reliability that are important for business data and mission-critical applications.
MySQL includes components such as the `InnoDB` storage engine that adhere closely to the ACID model so that data is not corrupted and results are not distorted by exceptional conditions such as software crashes and hardware malfunctions.
When you rely on ACID-compliant features, you do not need to reinvent the wheel of consistency checking and crash recovery mechanisms. In cases where you have additional software safeguards, ultra-reliable hardware, or an application that can tolerate a small amount of data loss or inconsistency, you can adjust MySQL settings to trade some of the ACID reliability for greater performance or throughput.

- ***A***: atomicity.
- ***C***: consistency.
- ***I***: isolation.
- ***D***: durability.

**Atomicity**

The ***atomicity*** aspect of the ACID model mainly involves `InnoDB` [transactions](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_transaction).

**Consistency**

The ***consistency*** aspect of the ACID model mainly involves internal `InnoDB` processing to protect data from crashes.

**Isolation**

The ***isolation*** aspect of the ACID model mainly involves `InnoDB` [transactions](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_transaction), in particular the isolation level that applies to each transaction.
