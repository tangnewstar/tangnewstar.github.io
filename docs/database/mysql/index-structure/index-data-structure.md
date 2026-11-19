---
title: 索引数据结构
---

# 索引数据结构

MySQL 官方将索引定义为用于**快速查找具有特定列值的行**的结构：没有索引时，MySQL 必须从表的第一行开始顺序扫描；有了索引，可以快速定位数据位置，而不必读取全部数据行。

索引的底层数据结构主要有三类：**B+树（B-Tree）**、**哈希表（Hash）** 和用于全文检索的**倒排列表（Inverted List）**。官方说明，MySQL 中大多数索引（`PRIMARY KEY`、`UNIQUE`、`INDEX`、`FULLTEXT`）存储在 B-tree 中；空间索引使用 R-tree；`MEMORY` 表支持哈希索引；`InnoDB` 的 `FULLTEXT` 索引使用倒排列表。

## B+树索引

B+树是一种平衡多路搜索树，是 MySQL 中最主流的索引结构（InnoDB 和 MyISAM 的默认索引结构）。

**结构特点**

- 数据都存储在**叶子节点**，叶子节点之间通过链表相连形成有序链表，适合范围查询。
- **非叶子节点**只存储键值和指针，用于快速定位叶子节点，不存储实际数据行。
- 树高通常很低，千万级数据下 3~4 层即可定位，查询通常只需 3~4 次 I/O。

**InnoDB 的 B+树索引**

- **聚簇索引（Clustered Index）**：以主键构建的 B+树，叶子节点存储完整行数据。InnoDB 表必有聚簇索引，未显式定义主键时会选择唯一非空索引，否则生成隐藏的 `DB_ROW_ID`。
- **二级索引（Secondary Index）**：叶子节点存储索引列值与对应主键值，查询非索引列时需要**回表**。

**官方列出的适用场景**

- `WHERE` 条件匹配：`=`、`>`、`>=`、`<`、`<=`、`BETWEEN`。
- 前缀匹配：`LIKE 'Patrick%'可以使用索引`，`LIKE '%Patrick%'` 无法使用。
- 多列索引的**最左前缀**：`(col1, col2, col3)` 可用于 `(col1)`、`(col1, col2)`、`(col1, col2, col3)`。
- 连接查询：连接列类型与大小一致时更高效。
- `MIN()` / `MAX()` 优化：前置索引部分为常量条件时。
- `ORDER BY`、`GROUP BY`：排序或分组字段符合索引最左前缀时。
- **覆盖索引**：查询列都包含在索引中时，直接从索引树返回结果，避免回表。

**限制**

- 小表或需要访问大部分行的查询中，顺序扫描通常比索引更快。
- 连接列类型、大小或字符集不一致会导致索引失效。

## 哈希索引

基于哈希表实现，通过哈希函数将键值映射到哈希桶。

- MySQL 中 `MEMORY` 存储引擎支持哈希索引；`InnoDB` 不支持用户显式创建哈希索引，但提供**自适应哈希索引**。
- 等值查询（`=`、`<=>`）非常快，查询复杂度接近 O(1)。
- 键值无序，**不支持范围查询、排序、最左前缀匹配**；也无法估算两个值之间的行数。

**InnoDB 自适应哈希索引**

官方描述：自适应哈希索引使 InnoDB 在具有合适的工作负载组合、且缓冲池内存充足的系统上，表现得更像内存数据库，同时不牺牲事务特性与可靠性。

- 由 `innodb_adaptive_hash_index` 控制，InnoDB 监控索引搜索，按需为经常访问的索引页构建哈希索引，使用索引键的**前缀**，前缀长度任意。
- 表几乎完全放入主内存时，哈希索引通过直接查找（把索引值变成指针）加速查询。
- 高并发连接或大量 JOIN 的工作负载下，访问自适应哈希索引可能成为**争用来源**；带 `LIKE` 和 `%` 通配符的查询通常无法受益。
- 是否开启应通过真实负载评估。

## 全文索引

全文索引是 `FULLTEXT` 类型的索引，用于在文本列中进行关键词检索。

- 只有 `InnoDB` 和 `MyISAM` 表支持，且只能创建在 `CHAR`、`VARCHAR`、`TEXT` 列上。
- InnoDB 的全文索引使用**倒排列表（inverted lists）**实现，而非 B+树。
- 查询语法：`MATCH (col1, col2, ...) AGAINST (expr [search_modifier])`。
- 检索模式：
    - **自然语言模式**（默认）：搜索字符串按自然短语解释，适用停用词列表。
    - **布尔模式**：支持操作符，指定词必须出现、必须不出现或权重高低。
    - **查询扩展模式**：先用原词搜索，再从最相关结果中提取词进行二次搜索。
- MySQL 内置 `ngram` 全文解析器支持中文、日文、韩文（CJK）分词；另有 MeCab 插件支持日文。
- 停用词（stopword）不会进入索引，因此不会匹配。
- 大批量导入建议：先导入没有 `FULLTEXT` 索引的表，导入完成后再创建全文索引，速度更快。

## 三类索引对比

| 索引类型 | 底层结构 | 等值查询 | 范围查询 | 排序 | 全文检索 | 主要引擎 |
|---|---|---|---|---|---|---|
| B+树 | 平衡多路树 + 有序叶子链表 | 支持 | 支持 | 支持 | 不支持 | InnoDB、MyISAM |
| 哈希 | 哈希表 | 极快 | 不支持 | 不支持 | 不支持 | MEMORY；InnoDB 自适应哈希 |
| 全文 | InnoDB 倒排列表 | 不适用 | 不适用 | 不适用 | 支持 | InnoDB、MyISAM |

## 参考

- [MySQL 8.0 Reference Manual · 10.3.1 How MySQL Uses Indexes](https://dev.mysql.com/doc/refman/8.0/en/mysql-indexes.html)
- [MySQL 8.0 Reference Manual · 10.3.9 Comparison of B-Tree and Hash Indexes](https://dev.mysql.com/doc/refman/8.0/en/index-btree-hash.html)
- [MySQL 8.0 Reference Manual · 14.9 Full-Text Search Functions](https://dev.mysql.com/doc/refman/8.0/en/fulltext-search.html)
- [MySQL 8.0 Reference Manual · 17.5.3 Adaptive Hash Index](https://dev.mysql.com/doc/refman/8.3/en/innodb-adaptive-hash.html)
