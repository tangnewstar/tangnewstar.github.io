# 查询解析与优化

MySQL 查询解析、执行计划与优化相关的内容。

一条 SQL 从文本到结果，依次经过**解析 → 优化 → 执行**：

- [查询解析器](query-parser.md)：把 SQL 文本解析为解析树，并做语义与权限检查
- [优化器](optimizer.md)：基于成本模型与统计信息选择执行方案
- [执行计划](execution-plan.md)：用 `EXPLAIN` 查看并分析优化器的选择

### 番外篇

- [番外篇：index Cardinality](bonus-cardinality.md)
