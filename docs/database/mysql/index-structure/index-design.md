---
title: 索引设计原则与失效场景
---

# 索引设计原则与失效场景

本文整合索引的设计原则，以及导致索引失效的常见查询场景。

## 索引设计原则

- 数据量大查询繁琐的表建立索引
- 常作为查询条件 where、排序 sort 和分组 group 操作的字段建立索引
- 尽量选择区分度高的列作为索引，尽量建立唯一索引。区分度越高，使用索引的效率越高
- 字符串类型的字段如果长度过长 varchar(5000)、text、longtext，可以根据字段特点建立前缀索引
- 尽量使用联合索引，查询时联合索引可以覆盖索引节省存储空间，避免回表提高效率
- 控制索引数量，索引越多维护索引结构的代价越大，影响增删改的效率
- 索引列不能存储 null，建表时约束 not null。优化器知道每列是否包含 null 值，可以更好地确定使用哪个索引有效查询

## 索引失效的常见情况

即使创建了索引，某些查询条件会导致 mysql 放弃使用索引，转而进行全表扫描，这就是索引失效。

常见原因：

- 索引列被函数修改或参与计算
- 使用 NOT ≠ <> 等否定运算
- 使用 OR 连接非索引列（如果 OR 条件中有列没有索引，可能会全表扫描）
- 字符串不加引号导致类型隐式转换，导致索引失效（例如字符串列用数字来查询）
- 联合索引不满足最左匹配原则

```sql
-- 生效：使用最左前缀 a
SELECT * FROM user WHERE a = 1;

-- 生效：使用 a + b
SELECT * FROM user WHERE a = 1 AND b = 2;

-- 失效：跳过 a，直接使用 b
SELECT * FROM user WHERE b = 2;

-- 失效：a 是范围查询，后续 b 失效
SELECT * FROM user WHERE a > 1 AND b = 2;
```

- LIKE 以通配符 % 开头

```sql
-- 失效：% 开头
SELECT * FROM user WHERE name LIKE '%Tom';

-- 生效：% 结尾
SELECT * FROM user WHERE name LIKE 'Tom%';
```

- 使用 IS NOT NULL

```sql
-- 可能失效
SELECT * FROM user WHERE age IS NOT NULL;
```
