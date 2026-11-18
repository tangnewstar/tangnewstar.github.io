---
title: 索引失效的常见情况
---

# 索引失效的常见情况

即使创建了索引，某些查询条件会导致mysql放弃使用索引，转而进行全表扫描，这就是索引失效。

常见原因：

- 索引列被函数修改或参与计算
- 使用NOT ≠ <> 等否定运算
- 使用OR连接非索引列（如果OR条件中有列没有索引，可能会全表扫描）
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

- LIKE以通配符%开头

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
