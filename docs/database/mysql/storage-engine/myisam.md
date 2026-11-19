---
title: MyISAM
---

# MyISAM

MyISAM 是 MySQL 的经典存储引擎，基于更早期的 ISAM 引擎并扩展了许多特性。在 MySQL 8.0 中，**InnoDB 是默认存储引擎**，MyISAM 需要显式指定 `ENGINE = MYISAM` 才会使用。官方文档将 MyISAM 定位为：表占用空间小、适合**只读或读多写少**的负载（如 Web、数据仓库场景）。

## 核心特性

| 特性 | MyISAM 支持情况 |
|---|---|
| B-tree 索引 | 支持 |
| 全文索引 Full-text | 支持 |
| 空间数据类型与空间索引 | 支持 |
| 压缩数据 | 支持（压缩后为只读） |
| 索引缓存 | 支持 |
| 数据缓存 | 不支持 |
| 锁粒度 | **表级锁** |
| 事务 / MVCC / 外键 | 不支持 |
| 聚簇索引 / Hash 索引 | 不支持 |
| 分区（MySQL 8.0） | 不支持 |
| 存储上限 | 256TB |

## 文件结构

官方说明：MySQL 8.0 中每张 MyISAM 表由两个文件存储，表定义保存在 MySQL 数据字典中：

- **`.MYD`**（MYData）：数据文件
- **`.MYI`**（MYIndex）：索引文件

文件名以表名开头，通过扩展名区分。可用 `DATA DIRECTORY` 和 `INDEX DIRECTORY` 把数据文件与索引文件放到不同目录或物理设备。索引文件中有标志位记录表是否被正确关闭，配合 `myisam_recover_options` 可在打开表时自动检查并修复异常关闭造成的问题；也可用 `mysqlcheck` 或 `myisamchk` 手动检查修复。

!!! note "关于 .frm 文件"
    旧版本 MySQL 中表定义存放在 `.frm` 文件；MySQL 8.0 起表定义改存于数据字典，官方文档已不再把 `.frm` 列为 MyISAM 表文件结构的一部分。

## 表级锁与并发插入

MyISAM 使用**表级锁**：写操作通常锁住整张表，因此不适合高并发写场景。相比 InnoDB 的行级锁，其并发写入能力较弱。

但 MyISAM 支持一种有限的**并发插入（concurrent inserts）**：如果数据文件中间没有空闲块，可以在其他线程读取同一张表的同时插入新行。删除行或变长更新会造成空闲块，当空闲块被重新用尽后，插入可恢复并发。

## 非事务安全

MyISAM **不支持事务、MVCC 与外键**，不属于事务安全型存储引擎，不能提供提交、回滚、外键约束与事务级一致性保障。如果应用依赖：

- `COMMIT` / `ROLLBACK`
- 事务隔离与 MVCC
- 外键约束
- 崩溃后的事务级一致性恢复

则不应选择 MyISAM。这也是 InnoDB 成为默认引擎、被推荐用于绝大多数场景的根本原因。

## 全文索引与空间索引

MyISAM 原生支持全文索引与空间数据类型/空间索引，在 InnoDB 支持这些特性之前（全文索引自 MySQL 5.6、空间索引自 MySQL 5.7 起支持），这是 MyISAM 的重要优势。如今两者在这些能力上已趋同，这一差异不再是主要选型理由（见 [索引数据结构](../index-structure/index-data-structure.md)）。

## 压缩表

MyISAM 支持用 `myisampack` 压缩表以节省空间，但**压缩后的表是只读的**。适合归档、只读数据集等对空间敏感但无需更新的场景。

## 其他特性

- 数据值以低字节优先存储，在机器与操作系统间具有二进制可移植性；数值索引键以高字节优先存储以改善压缩。
- 单表最多 64 个索引，每个索引最多 16 列，最大键长度 1000 字节。
- 每个表允许一个 `AUTO_INCREMENT` 列；支持对 `BLOB`、`TEXT` 列建索引；索引列允许 `NULL`。
- 每个字符列可使用不同字符集。

## MyISAM 与 InnoDB 对比

| 特性 | MyISAM | InnoDB |
|---|---|---|
| 事务 | 不支持 | 支持（ACID） |
| 锁粒度 | 表级锁 | 行级锁 |
| MVCC | 不支持 | 支持 |
| 外键 | 不支持 | 支持 |
| 聚簇索引 | 不支持 | 支持 |
| 崩溃恢复 | 不支持 | 支持 |
| 数据缓存 | 不支持（仅索引缓存） | 支持（Buffer Pool） |
| 全文索引 | 支持 | 支持（5.6+） |
| 空间索引 | 支持 | 支持（5.7+） |
| 存储上限 | 256TB | 64TB |
| 适用场景 | 只读 / 读多写少、归档 | 通用场景，绝大多数业务表 |

社区经验与官方建议一致：**除非有明确理由，优先使用 InnoDB**。MyISAM 如今主要用于遗留系统兼容、只读归档表，或某些读密集且完全不需要事务的特殊表。

## 相关文档

- [InnoDB Overview](innodb-overview.md)
- [ACID 实现原理](../transaction/acid.md)
- [锁机制](../concurrency/locking.md)

## 参考

- [MySQL 8.0 Reference Manual · 18.2 The MyISAM Storage Engine](https://dev.mysql.com/doc/refman/8.0/en/myisam-storage-engine.html)
- [MySQL 8.0 Reference Manual · 18 Alternative Storage Engines](https://dev.mysql.com/doc/refman/8.0/en/storage-engines.html)
