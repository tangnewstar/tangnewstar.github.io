---
title: 优化器
---

# 优化器

查询优化器（Query Optimizer）是 MySQL 决定“**怎么执行查询**”的组件。同一条 SQL 往往有多种执行方式，优化器负责在这些候选方案中选择它认为成本较低的一个，并生成执行计划交给执行器。

MySQL 官方把优化器控制能力归纳为几类：影响查询计划评估的**系统变量**、**可切换优化**（`optimizer_switch`）、**优化器提示与索引提示**、**优化器成本模型**，以及**优化器统计信息（直方图）**。理解优化器，就是理解它“如何估算、如何选、我们如何影响它”。

## 优化器做什么

对于一条查询，优化器需要决定：

- **访问路径**：全表扫描还是走索引？走哪个索引？
- **连接顺序**：多表连接时，先读哪张表、后读哪张表？
- **连接方式**：嵌套循环、hash join 还是其他策略？
- **子查询处理**：物化、半连接转换（semijoin）等。
- **排序与分组**：是否利用索引有序性避免 `filesort`，是否使用临时表。

优化器选择的结果，就是可以用 `EXPLAIN` 看到的**执行计划**（见 [执行计划](execution-plan.md)）。

## 成本模型：如何估算

官方说明：为了生成执行计划，优化器使用一个**基于成本（cost）的模型**，估算查询执行中各类操作的成本，据此在不同方案间做选择。成本来源有两类：

1. **编译内置的默认成本常量**；
2. **可配置的成本估算表**，位于 `mysql` 系统库：
    - `server_cost`：通用服务器操作成本，如 `row_evaluate_cost`（评估记录条件的成本）、`key_compare_cost`（键比较成本）、临时表创建与行处理成本。
    - `engine_cost`：存储引擎相关成本，如 `io_block_read_cost`（从磁盘读块成本）、`memory_block_read_cost`（从内存缓冲区读块成本）。

运行机制：服务器启动时把成本表读入内存；某成本项的 `cost_value` 非 `NULL` 时覆盖内置默认值，`NULL` 时使用默认值。修改成本表后需执行 `FLUSH OPTIMIZER_COSTS` 让服务器重新读取；成本表是实例级的，变更不会复制到副本。

官方调整建议：通常最值得关注的是 `io_block_read_cost` 与 `memory_block_read_cost`，可先尝试**加倍或减半**再测量效果。例如当 `io_block_read_cost` 高于 `memory_block_read_cost` 时，优化器更偏好读取已在内存中的数据。

## 优化器统计：cardinality 与直方图

优化器的估算依赖统计信息。前面介绍过索引的 **cardinality**（见 [番外篇：index Cardinality](bonus-cardinality.md)）——优化器用它判断某个索引能过滤掉多少行。

MySQL 8.0 还维护关于列值的**直方图（histogram）统计**，保存在数据字典表 `column_statistics` 中。该表不能直接访问，用户通过 `INFORMATION_SCHEMA.COLUMN_STATISTICS` 查看，并用 `ANALYZE TABLE` 管理。当优化器认为索引本应被使用却未被使用时，官方建议先用 `ANALYZE TABLE` 更新统计信息。

## 如何影响优化器

当优化器的选择不理想时，官方提供多种控制手段：

### 1. 系统变量

通过影响查询计划评估的系统变量控制优化器行为。

### 2. 可切换优化：`optimizer_switch`

通过 `optimizer_switch` 打开或关闭特定优化策略，例如 Index Merge、semijoin 等：

```sql
SET SESSION optimizer_switch = 'index_merge=off';
```

### 3. 优化器提示与索引提示

- **Optimizer Hints**：在 SQL 中用注释形式给出提示，如 `/*+ MAX_EXECUTION_TIME(1000) */`。
- **Index Hints**：用 `USE INDEX`、`FORCE INDEX`、`IGNORE INDEX` 影响索引选择。

```sql
SELECT * FROM users FORCE INDEX (idx_age) WHERE age > 18;
```

### 4. 连接顺序提示

官方在 EXPLAIN 文档中提到：如果发现连接顺序不理想，可用 `STRAIGHT_JOIN` 让优化器按 `SELECT` 语句中表名的出现顺序连接。

### 5. 更新统计信息

用 `ANALYZE TABLE` 更新表的统计信息（如 cardinality、直方图），让优化器基于最新数据做估算。

## 优化器不是万能的

- 优化器基于**估算**做决策，估算可能不准；行数、数据分布、硬件状态都会影响结果。
- 官方明确：`EXPLAIN ANALYZE` 可对比优化器预期与实际执行，用于发现估算偏差（见 [执行计划](execution-plan.md)）。
- 调成本模型或加提示应作为最后手段，先检查索引设计与统计信息是否到位。

## 相关文档

- [查询解析器](query-parser.md)
- [执行计划](execution-plan.md)
- [索引设计](../index-structure/index-design.md)

## 参考

- [MySQL 8.0 Reference Manual · 10.9 Controlling the Query Optimizer](https://dev.mysql.com/doc/refman/8.0/en/controlling-optimizer.html)
- [MySQL 8.0 Reference Manual · 10.9.5 The Optimizer Cost Model](https://dev.mysql.com/doc/refman/8.0/en/cost-model.html)
