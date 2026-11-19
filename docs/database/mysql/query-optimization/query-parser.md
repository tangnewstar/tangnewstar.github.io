---
title: 查询解析器
---

# 查询解析器

查询解析器（Parser）是 SQL 语句进入服务器后的第一道处理环节，负责把 SQL 文本转化为服务器可以理解的内部结构。官方虽未设独立的“解析器”章节，但从预处理语句的说明中可以清楚看到解析开销的存在：使用占位符的预处理语句能**减少每次执行时解析语句的开销**（Less overhead for parsing the statement each time it is executed），这正说明解析是每次执行 SQL 都要经历的一步。

## 一条 SQL 的处理流程

从客户端发出 SQL 到拿到结果，大致经过以下阶段：

```text
SQL 文本
  → 词法分析（Tokenizer）：拆分为 token
  → 语法分析（Parser）：按语法规则构建解析树
  → 预处理（Preprocessor）：语义检查、名称解析、权限检查
  → 优化器（Optimizer）：生成并选取执行计划（见《优化器》）
  → 执行器（Executor）：调用存储引擎执行（见《执行计划》）
  → 返回结果
```

解析器主要承担前两步，第三步通常称为预处理，二者紧密衔接。

## 词法分析：拆分为 token

词法分析把连续的 SQL 字符串切分为有意义的词法单元（token），例如关键字、标识符、字面量、运算符和分隔符：

```sql
SELECT id FROM users WHERE age > 18;
```

会被拆成大致如下的 token 序列：

```text
SELECT | id | FROM | users | WHERE | age | > | 18 | ;
```

这一阶段会识别字符串引号、注释、数字格式等；如果字符串没有正确闭合、出现非法字符，会在这一步报错。

## 语法分析：构建解析树

语法分析器依据 MySQL 的 SQL 语法规则，把 token 序列组织成一棵**解析树（parse tree）**，检查语句结构是否符合语法：

- 关键字顺序是否正确（`SELECT ... FROM ... WHERE ...`）
- 子句是否完整、括号是否配对
- 表达式是否合法

如果 SQL 写错关键字或结构，例如 `SELEC id FROM users`，会在语法分析阶段报错（如 `You have an error in your SQL syntax`）。只有语法合法的语句才能进入后续处理。

## 预处理：语义检查

解析树生成后，预处理阶段做语义层面的检查，把“语法正确”进一步变为“语义有效”：

- **名称解析**：确认表、列、别名、函数是否存在，是否歧义。
- **权限检查**：确认当前用户是否有相应操作权限。
- **语义规则**：例如聚合函数与 `GROUP BY` 的合法性、列是否在正确的上下文中使用。
- **视图展开**：把视图引用转换为对应的查询表达式。

语法错误与语义错误的区别：`SELECT * FORM users` 是语法错误（关键字拼错）；`SELECT no_such_col FROM users` 语法正确但列不存在，是语义错误。

## 解析开销与预处理语句

官方说明，服务端预处理语句（Prepared Statements）利用高效的客户端/服务端二进制协议，带来两个收益：

- **减少每次执行时解析语句的开销**（Less overhead for parsing the statement each time it is executed）。
- **降低 SQL 注入风险**（参数值可以包含未转义的引号和分隔符）。

流程是：先 `PREPARE` 一条带 `?` 占位符的 SQL 模板，执行时用 `EXECUTE ... USING` 绑定参数值，同一模板多次执行而不必每次重新解析相同的 SQL 文本。官方同时指出：如果语句引用的表或视图**元数据发生变化**，MySQL 会检测到并在下一次执行时**自动重新准备**该语句，因此解析结果与元数据状态是关联的。

```sql
PREPARE stmt FROM 'SELECT * FROM users WHERE age > ?';
SET @age = 18;
EXECUTE stmt USING @age;
DEALLOCATE PREPARE stmt;
```

## 小结

- 解析器解决的是“**SQL 写得对不对**”：先拆 token，再按语法构建解析树，然后做语义与权限检查。
- 解析发生在优化之前；解析通过的查询才会交给优化器生成执行计划。
- 预处理语句通过“一次准备、多次执行”避免重复解析，是减少解析开销的常规手段。

## 相关文档

- [优化器](optimizer.md)
- [执行计划](execution-plan.md)
- [番外篇：index Cardinality](bonus-cardinality.md)

## 参考

- [MySQL 8.0 Reference Manual · 15.5 Prepared Statements](https://dev.mysql.com/doc/refman/8.0/en/sql-prepared-statements.html)
