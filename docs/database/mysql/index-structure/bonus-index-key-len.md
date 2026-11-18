---
title: 番外篇：索引长度计算
---

# 番外篇：索引长度计算

key_len = （类型长度）+ (没有 NOT NULL)

| 场景 | 类型 | 长度 |
| --- | --- | --- |
| 没有 NOT NULL | ALL | 1 byte |
| 长度固定 | tinyint | 1 byte |
|  | int | 4 byte |
|  | bigint | 8 bytes |
|  | date | 3 bytes |
|  | datetime | 5 bytes |
|  | char(n) | n * len(char) |
| 长度变化 | varchar(n) | n * len(char) + 2 bytes |

不同字符集 len(char) 不同：

| 字符集 | len(char) |
| --- | --- |
| latin1 | 1 byte |
| gbk | 2 bytes |
| utf8 | 3 bytes |
| utf8mb4 | 4 bytes |

example：

- utf8mb4
    - name varchar(100) not null `CHARACTER SET utf8mb4`
    - key_len = 100 * 4 + 2 = 402
- name varchar(100) not null `CHARACTER SET utf8`
    - key_len = 100 * 3 + 2 = 302
