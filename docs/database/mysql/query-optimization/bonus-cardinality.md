---
title: 番外篇：index Cardinality
---

# 番外篇：index Cardinality

### 介绍 mysql index cardinality

index cardinality 指的是存储在具体的索引列的值的不重复性。由于是基于统计生成，因此该值不一定准确。

对于给定的查询语句，查询优化器（query optimizer）使用 index cardinality 去生成可选的执行计划（query plan）。同时在 join 操作中会使用 index cardinality 决定是否使用该索引。

如果查询优化器选择了低 cardinality 的索引，可能比不使用索引全表扫描更有效率。

cardinality 值的高低是相对的，通常和表的总行数比较来看。

### 查看 index cardinality

使用 `show index` 查看 index cardinality，如下示例 Orders 订单表：

```sql
mysql> SHOW INDEX FROM orders;
+--------+------------+----------------+--------------+----------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+
| Table  | Non_unique | Key_name       | Seq_in_index | Column_name    | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment | Visible |
+--------+------------+----------------+--------------+----------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+
| orders |          0 | PRIMARY        |            1 | orderNumber    | A         |         326 |     NULL |   NULL |      | BTREE      |         |               | YES     |
| orders |          1 | customerNumber |            1 | customerNumber | A         |          98 |     NULL |   NULL |      | BTREE      |         |               | YES     |
+--------+------------+----------------+--------------+----------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+
2 rows in set (0.01 sec)
```

订单表有两个索引，主键索引和普通索引 customerNumber：

- orderNumber 列（主键）的 index cardinality 为 326，表的总行数是 326
- customerNumber 列的 index cardinality 为 98，意味着这一列有 98 个不重复值

可以验证一下 customerNumber 列的去重数：

```sql
SELECT COUNT(DISTINCT customerNumber) FROM orders;

+--------------------------------+
| COUNT(DISTINCT customerNumber) |
+--------------------------------+
|                             98 |
+--------------------------------+
1 row in set (0.00 sec)
```

如果一个列的 cardinality 值高，可能是作为查询条件的理想选择。例如查询某个客户的订单，因为 cardinality 相对高，以 customerNumber 为索引会提升效率：

```sql
SELECT * FROM orders WHERE customerNumber = 103;
```

### 总结

- index cardinality 是具体索引列值不重复的度量
- mysql 使用 index cardinality 去优化数据库性能和查询效率
