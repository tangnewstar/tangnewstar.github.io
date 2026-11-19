---
title: MGR、InnoDB Cluster 与自动切换
---

# MGR、InnoDB Cluster 与自动切换

传统的[主从复制](master-slave-replication.md)多为主库故障后手动提升从库，或配合第三方工具切换。MySQL 官方提供了原生的高可用方案：**MGR（Group Replication）** 负责组内的复制与一致性，**InnoDB Cluster** 将其封装为完整集群，**MySQL Router** 则为应用提供透明的自动切换入口。

## MGR（MySQL Group Replication）

官方定义：

> MySQL Group Replication enables you to create elastic, highly-available, fault-tolerant replication topologies.

即 MGR 使你可以创建**弹性、高可用、容错**的复制拓扑。它以 MySQL Server 插件形式提供，需要在每个希望加入组的实例上配置。

### 核心机制

- **组成员服务（Group Membership Service）**：维护组成员视图，保证任意时刻所有服务器看到的组视图一致；成员加入或离开时视图自动更新。
- **故障检测（Failure Detection）**：成员意外离开时自动检测，并通知组内其他成员更新视图。
- **分布式恢复（Distributed Recovery）**：成员加入或重新加入组时自动进行数据恢复。
- **GTID 与监控**：通过 `replication_group_members`、`replication_group_member_stats` 等表查看成员状态与复制统计。

### 复制模式

- **单主模式（Single-Primary）**：组内只有一个 Primary 接受写操作，其他为 Secondary。Primary 故障时通过**自动选主**选出新主，是默认模式。
- **多主模式（Multi-Primary）**：所有服务器都可接受更新，即使并发写入。对应用设计与冲突处理要求更高。

### 局限

官方特别强调：

> Group Replication does not have an inbuilt method to do this.

即 MGR **不内置客户端故障转移**。组成员视图更新、故障检测、单主模式重新选主都是自动的，但应用连接到故障节点时，需要外部组件（connector、负载均衡器、MySQL Router 等）重定向连接。

| 能力 | 是否自动 |
|---|---|
| 组成员视图更新 | 自动 |
| 故障检测 | 自动 |
| 单主模式重新选主 | 自动 |
| 应用/客户端连接切换 | 需借助外部组件 |

## InnoDB Cluster

InnoDB Cluster 是 MySQL 官方的高可用解决方案，组合了三项技术：

| 层级 | 组件 | 作用 |
|---|---|---|
| 应用接入层 | MySQL Router | 连接路由、读写分离、故障转移感知 |
| 管理控制层 | MySQL Shell（AdminAPI） | 集群创建、配置、运维、状态查看 |
| 数据复制层 | Group Replication + MySQL Server | 数据复制、一致性控制、成员管理、选主 |

要点：

- 通过 MySQL Shell 的 AdminAPI，可轻松配置和管理**至少三个** MySQL 服务器实例组成的 InnoDB Cluster。
- 每个实例运行 Group Replication，提供复制机制与内置容错；默认单主模式下只有一个读写 Primary，其余为 Secondary。
- MySQL Router 可根据部署的集群自动配置，把客户端应用**透明**连接到服务器实例；实例意外故障时集群自动重新配置。
- **InnoDB ClusterSet** 可将主 InnoDB Cluster 与一个或多个位于备用位置（如其他数据中心）的副本链接，提供容灾能力。

### 自动切换流程

完整的自动切换由 Group Replication 与 MySQL Router 协作完成：

1. **故障检测**：Group Replication 通过组内通信检测节点不可用。
2. **成员剔除**：故障节点从当前成员视图中移除。
3. **多数派判断**：只有存活节点构成多数派（quorum）时才能继续写；失去多数派时阻止写入，避免脑裂。
4. **自动选主**：单主模式下在剩余健康节点中选出新 Primary。
5. **路由切换**：MySQL Router 感知集群拓扑变化，将新连接路由到新 Primary。
6. **应用重连**：应用会经历短暂连接中断，建议实现连接重试。

## 自动切换工具

### MySQL Router

MySQL Router 是官方配套的透明路由组件：

- 为应用提供稳定连接入口，屏蔽后端拓扑变化。
- 将读写请求路由到当前 Primary，只读请求可分发到 Secondary。
- 感知集群拓扑变化，Primary 故障后把新连接转发到新主。
- 注意分工：**选主由 Group Replication 完成，Router 负责按拓扑变化重新路由连接**。

### 第三方工具

在传统主从复制架构中，也常使用第三方工具实现自动切换，例如：

- **MHA（Master High Availability）**：基于主从复制的主库故障切换工具。
- **Orchestrator**：拓扑发现与故障转移工具。
- **Keepalived / VIP**：配合漂移 IP 实现入口切换。

这些工具不属于 MySQL 官方 InnoDB Cluster 体系，适用于仍采用异步/半同步复制的场景。

## 小结

| 方案 | 复制底层 | 选主 | 客户端切换 |
|---|---|---|---|
| 主从复制 + 人工 | binlog 异步/半同步 | 手动 | 手动 |
| 主从复制 + 第三方工具 | binlog | 工具完成 | 工具 / VIP |
| MGR | Group Replication | 自动 | 需 MySQL Router 等 |
| InnoDB Cluster | Group Replication | 自动 | MySQL Router 自动路由 |

## 参考

- [MySQL 8.0 Reference Manual · 20 Group Replication](https://dev.mysql.com/doc/refman/8.0/en/group-replication.html)
- [MySQL Shell Documentation · Chapter 7 MySQL InnoDB Cluster](https://dev.mysql.com/doc/mysql-shell-excerpt/5.7/en/mysql-innodb-cluster.html)
- [MySQL 主从复制](master-slave-replication.md)
