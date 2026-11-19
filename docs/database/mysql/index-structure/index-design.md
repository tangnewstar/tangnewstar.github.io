---
title: 索引设计
---

# 索引设计

索引设计的核心矛盾是：**索引提升查询速度，但代价是占用磁盘空间、增加优化器选择成本、拖慢写入**（每次 `INSERT`、`UPDATE`、`DELETE` 都要维护索引）。MySQL 官方明确建议：不要因为某列“可能用于查询”就给它建索引，而应围绕实际查询，建立一组**最优索引集合**，在查询性能与维护成本之间取得平衡。

下面按“怎么建才对”的顺序给出设计原则，并说明每条背后的原因。

## 一、先看表与查询，再决定建不建

- **只为真正被查询使用的列建索引**。官方指出，提高 `SELECT` 性能的最佳方式是在查询条件涉及的列上建索引；反之，未被使用的索引只会浪费空间、干扰优化器、拖慢写入。
- **优先为 `WHERE`、`ORDER BY`、`GROUP BY`、连接条件中的列建索引**，这些是索引发挥作用的主要场景。
- **小表通常不需要索引**。官方说明：当查询需要访问表中大部分行时，顺序读取比通过索引读取更快（减少磁盘寻址），此时索引收益有限。
- **选择区分度高的列**。区分度越高，单次索引查找能过滤掉越多行。衡量方式是 cardinality / 总行数，越接近 1 越好（详见 [番外篇：index Cardinality](../query-optimization/bonus-cardinality.md)）。
- **控制索引数量**。索引越多，写操作维护索引的成本越高，优化器选择索引的判断成本也越高。

## 二、主键怎么建

在 InnoDB 中，主键不只是约束，更是**聚簇索引**，表数据按主键顺序物理组织。官方给出的选择规则与建议：

1. **每张表尽量显式定义主键**。有主键时 InnoDB 用它作聚簇索引；没有主键时退而选择第一个所有列都 `NOT NULL` 的 `UNIQUE` 索引；两者都没有时，InnoDB 会生成隐藏的 6 字节 `GEN_CLUST_INDEX`（按行 ID，随插入单调递增）。
2. **优先选最常被查询的列或列组合**。主键自带索引，且 InnoDB 中通过主键访问行非常快——索引查找直接定位到含行数据的页，大表上通常可节省一次磁盘 I/O。
3. **没有合适的自然主键时，用单独的自增列**。官方明确建议：对于很大、很重要、但没有明显主键候选的表，添加一个自增列作为主键；这个唯一 ID 也可作为行指针，用于外键关联。
4. **主键要短**。所有二级索引的叶子节点都会存一份主键值，主键越长，每个二级索引记录越大、占用空间越多。
5. **主键列必须 `NOT NULL`**，这本身就能让优化器利用 `NOT NULL` 优化。

## 三、联合索引怎么建

官方说明：一个多列索引最多可包含 16 列；合理设计列顺序后，**一个联合索引可以加速多种查询**。设计要点：

### 1. 遵循最左前缀原则

对索引 `(col1, col2, col3)`，可用于 `(col1)`、`(col1, col2)`、`(col1, col2, col3)` 的查找，但不能用于 `(col2)`、`(col2, col3)`。因此：

- **把最常作为查询入口的列放最左边**。
- **按查询组合的层级排列**，让一个索引覆盖多种查询。例如常见查询是：

```sql
WHERE user_id = ?
WHERE user_id = ? AND status = ?
WHERE user_id = ? AND status = ? AND created_at >= ?
```

则 `INDEX (user_id, status, created_at)` 可同时服务这三种查询。

### 2. 等值列在前，范围列在后

官方示例中，`WHERE last_name='Jones' AND first_name >= 'M' AND first_name < 'N'` 对应 `INDEX (last_name, first_name)`：等值过滤列在前，范围过滤列在后。这是因为范围列之后的列无法继续用于精确定位（联合索引中范围查询后续列失效）。

### 3. 经常单独查询的列，不要只放在联合索引后面

如果经常 `WHERE b = ?`，那么 `INDEX (a, b)` 无法有效用于该查询，需要另建以 `b` 为最左列的索引，如 `INDEX (b)` 或 `INDEX (b, c)`。不要指望一个联合索引覆盖所有列组合。

### 4. 一个联合索引可能优于多个单列索引

对 `WHERE col1 = ? AND col2 = ?`，`INDEX (col1, col2)` 可直接定位行；而两个独立单列索引，优化器只能选择 Index Merge 或先用过滤性更强的那个索引再回表过滤。

### 5. 索引列很宽时，考虑短哈希列

官方提到：如果联合索引非常宽，可以增加一个短的、近似唯一的哈希列并建索引，例如：

```sql
SELECT * FROM tbl_name
WHERE hash_col = MD5(CONCAT(val1, val2))
  AND col1 = val1 AND col2 = val2;
```

查询中仍保留原始列条件，以避免哈希冲突带来的误判。

## 四、索引列怎么建

- **字符串列过长（`VARCHAR(5000)`、`TEXT`、`BLOB`）时，用前缀索引**。语法 `col_name(N)` 只索引前 `N` 个字符/字节，可显著减小索引体积；索引 `BLOB`/`TEXT` 时**必须**指定前缀长度。
    - 注意：前缀长度上限按**字节**计算；`InnoDB` 在 `DYNAMIC`/`COMPRESSED` 行格式下上限为 3072 字节，`REDUNDANT`/`COMPACT` 下为 767 字节。
    - 前缀太短会降低过滤精度，MySQL 仍需回表检查剩余部分是否真正匹配。
    - 全文搜索应使用 `FULLTEXT` 索引，它总是索引整列，不支持列前缀。
- **尽量让索引覆盖查询**（覆盖索引）。如果查询所需的列都在索引中，MySQL 可直接从索引树返回结果，避免回表。
- **声明 `NOT NULL` 约束**。优化器知道列是否包含 `NULL`，能更好地判断哪个索引对查询有效。

## 五、建完要验证

索引不是建完就生效。官方建议用 `EXPLAIN` 检查优化器是否真正使用了索引，并定期用 `SHOW INDEX` 查看索引统计（cardinality）。如果优化器估计使用索引会访问大量行，它可能选择全表扫描——这时应重新评估索引是否有价值，而不是简单“加了索引就该变快”。

## 参考

- [MySQL 8.0 Reference Manual · 10.3 Optimization and Indexes](https://dev.mysql.com/doc/refman/8.0/en/optimization-indexes.html)
- [MySQL 8.0 Reference Manual · 10.3.2 Primary Key Optimization](https://dev.mysql.com/doc/refman/8.0/en/primary-key-optimization.html)
- [MySQL 8.0 Reference Manual · 10.3.5 Column Indexes](https://dev.mysql.com/doc/refman/8.0/en/column-indexes.html)
- [MySQL 8.0 Reference Manual · 10.3.6 Multiple-Column Indexes](https://dev.mysql.com/doc/refman/8.0/en/multiple-column-indexes.html)
- [MySQL 8.0 Reference Manual · 17.6.2.1 Clustered and Secondary Indexes](https://dev.mysql.com/doc/refman/8.0/en/innodb-index-types.html)
