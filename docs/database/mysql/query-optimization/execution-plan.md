---
title: 执行计划
---

# 执行计划

执行计划（Execution Plan）是优化器给出的“MySQL 将如何执行这条查询”的说明。查看执行计划的主要工具是 `EXPLAIN`。官方说明：`DESCRIBE`、`DESC` 与 `EXPLAIN` 在语法上是同义词，但实践中 `DESCRIBE` 多用于查看表结构，`EXPLAIN` 多用于获取查询执行计划。

通过执行计划，可以看到 MySQL 如何处理语句、如何连接表以及按什么顺序连接表，从而判断索引是否有效、连接顺序是否合理。

## EXPLAIN 的基本用法

```sql
EXPLAIN SELECT * FROM users WHERE age > 18;
```

`EXPLAIN` 适用于 `SELECT`、`DELETE`、`INSERT`、`REPLACE`、`UPDATE`（MySQL 8.0.19 起也支持 `TABLE`），并需要与被解释语句本身相同的权限。

其他用法：

```sql
-- 查看某个连接正在执行语句的计划
EXPLAIN FOR CONNECTION connection_id;

-- 实际运行语句并对比预期与实际（MySQL 8.0.18 起）
EXPLAIN ANALYZE SELECT * FROM users WHERE age > 18;
```

## 输出格式

官方支持多种输出格式：

| 格式 | 说明 |
|---|---|
| `TRADITIONAL` | 默认表格格式，适合快速查看 |
| `JSON` | 结构化输出，包含成本信息，适合程序化分析 |
| `TREE` | MySQL 8.0.16 起支持，树状结构更精确，是唯一显示 hash join 的格式 |
| `EXPLAIN ANALYZE` | 始终使用 `TREE` 格式，显示实际执行情况 |

```sql
EXPLAIN FORMAT=JSON SELECT ...;
EXPLAIN FORMAT=TREE SELECT ...;
```

## EXPLAIN 输出列

传统表格格式下各列含义（以 `SELECT` 为例，查询中使用的每张表返回一行，按 MySQL 读取表的顺序显示）：

| 列 | 含义 |
|---|---|
| `id` | `SELECT` 的标识符与顺序编号 |
| `select_type` | `SELECT` 类型，如 `SIMPLE`、`PRIMARY`、`UNION`、`SUBQUERY`、`DERIVED`、`MATERIALIZED` 等 |
| `table` | 该行对应的表，也可能是 `<unionM,N>`、`<derivedN>`、`<subqueryN>` |
| `type` | 访问类型（连接类型），见下文 |
| `possible_keys` | MySQL 可从中选择的索引；`NULL` 表示没有相关索引 |
| `key` | 实际使用的索引；`NULL` 表示未使用索引 |
| `key_len` | 实际使用的键长度，可判断复合索引用了几部分 |
| `ref` | 与索引比较的列或常量；`func` 表示函数结果 |
| `rows` | MySQL 认为需要检查的行数；对 `InnoDB` 是估算值 |
| `filtered` | 条件过滤后剩余行数的估计百分比；参与下一步连接约为 `rows × filtered` |
| `Extra` | 附加信息，如 `Using index`、`Using filesort`、`Using temporary` 等 |

## 访问类型 type：从好到差

官方给出的顺序：

```text
system > const > eq_ref > ref > fulltext > ref_or_null > index_merge >
unique_subquery > index_subquery > range > index > ALL
```

| 类型 | 含义 |
|---|---|
| `system` | 表只有一行，是 `const` 的特殊情况 |
| `const` | 最多一个匹配行，查询开始时读取一次，非常快；常见于主键或唯一索引与常量比较 |
| `eq_ref` | 对前面表的每行组合，从该表读取一行；除 `system`、`const` 外最好的连接类型 |
| `ref` | 读取所有具有匹配索引值的行，用于非唯一索引或最左前缀的 `=` / `<=>` 比较 |
| `range` | 使用索引检索指定范围，如 `BETWEEN`、`<`、`>`、`IN()` |
| `index` | 扫描索引树；若索引覆盖查询列，`Extra` 显示 `Using index`，否则接近全表扫描 |
| `ALL` | 完整表扫描，通常是最差的类型 |

优化查询时，至少应让 `type` 达到 `range` 级别，避免出现 `ALL`。

## Extra 中值得关注的值

| `Extra` 值 | 含义 |
|---|---|
| `Using index` | 覆盖索引，只从索引树读取列，无需回表——**好事** |
| `Using where` | 使用 `WHERE` 条件过滤行 |
| `Using index condition` | 索引条件下推，先检查索引条件再决定是否读完整行 |
| `Using filesort` | 需要额外排序过程，通常值得优化 |
| `Using temporary` | 需要创建临时表，常见于复杂 `GROUP BY`、`ORDER BY` |
| `Impossible WHERE` | `WHERE` 条件恒为假 |

## 用 EXPLAIN 优化查询

官方给出的思路：

1. **判断是否需要添加索引**：对比 `possible_keys` 与 `key`，看哪些索引本可却被未使用；关注 `key_len` 判断复合索引用了几部分，关注 `rows`、`filtered` 判断过滤效果。
2. **检查多表连接顺序**：`EXPLAIN` 会显示连接顺序；不理想时可考虑 `STRAIGHT_JOIN`。
3. **索引未按预期使用时更新统计信息**：用 `ANALYZE TABLE` 更新表统计（如 cardinality）。
4. **用 `EXPLAIN ANALYZE` 对比预期与实际**：它会真实运行语句，显示每个 iterator 的估算成本、预计行数、实际耗时、实际行数与循环次数，据此找出耗时最高的环节、估算偏差，以及是否缺少索引或统计过期。

```text
-- EXPLAIN ANALYZE 输出示例中的关键字段
actual time=0.023...  rows=...  loops=...
```

!!! note "EXPLAIN ANALYZE 会真实执行语句"
    官方提醒：`EXPLAIN ANALYZE` 会真实运行被解释的语句，因此不要对不希望实际执行的写语句随意使用。MySQL 8.0.20 起可用 `KILL QUERY` 或 Ctrl-C 终止。

## 相关文档

- [查询解析器](query-parser.md)
- [优化器](optimizer.md)
- [番外篇：index Cardinality](bonus-cardinality.md)

## 参考

- [MySQL 8.0 Reference Manual · 15.8.2 EXPLAIN Statement](https://dev.mysql.com/doc/refman/8.0/en/explain.html)
- [MySQL 8.0 Reference Manual · 10.8.2 EXPLAIN Output Format](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html)
