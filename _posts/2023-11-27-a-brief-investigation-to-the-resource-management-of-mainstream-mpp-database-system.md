---
layout: post
title: "A Brief Investigation to the Resource Management of Mainstream MPP Database System"
date: 2023-11-27
categories: blog
permalink: /:categories/:year/:month/:title.html
---

## 1. 资源管理的定义和理解

我认为 MPP 数据库的资源管理能力是层层递进的，下面列出的 1 到 3 能力逐渐变强、功能越来越完善：
1. 资源隔离  \\
   1.1 资源限制形式：软隔离 / 硬隔离。  \\
   1.2 支持 SQL 分类器：自动路由到对应资源组。
2. 资源保护：并发限制与排队。
3. 优先级调整：资源组之间的优先级？资源组内部的优先级？

对于 1：
- 资源隔离是最基本的功能，例如资源组 A 占 30% 资源，B 占 70% 资源等，对资源的隔离限制还分为软隔离和硬隔离，软隔离是指当 A 满载而 B 空载的时候，A 能使用 100% 资源，即允许“超用”；硬隔离是指不管 B 是否有负载，A 最多只能使用 30% 的资源。
- 既然已经隔离出资源，就要解决“什么 SQL 跑到什么资源组上？”的问题，例如资源组 A 给 SELECT 语句，资源组 B 给 INSERT 语句，又或者资源组 A 给 user1，资源组 B 给 boss1 等。一些产品称这个功能为“分类器”。

对于 2：
- 资源隔离做好后，资源组之间的 SQL 隔离完成了，但是资源组内部还是会打架。对于一个资源组，如果并发太高会导致资源组内部的 SQL 一起变慢或卡住。因此要在资源组内部做并发限制或排队，两种方式都很好理解，限制一个资源组内的最大 QPS，超过则排队或报错。这部分能力主要是为了对资源保护。

对于 3：
- 这是更进一步的能力，提供给用户更灵活的控制权限。资源组之间的优先级之分，资源组内部的 SQL 的优先级之分等等，这些都是很细粒度的功能。优先级也可以翻译为重要性（Importance），Azure Synapse 对 Importance [1] 做了很多解释，提供丰富的配置。StarRocks（SR）虽然没有优先级这一说法，但是单独提供了名为 short_query 的资源组 [5] 专门给短查询用，这也是一种优先级的体现。

所以，资源管理的定义是什么？我觉得 Greenplum 文档 [6] 里的描述是最优雅和准确的：
> Greenplum Database provides features to help you prioritize and allocate resources to queries according to business requirements and to prevent queries from starting when resources are unavailable.

`accoring to business requirements`强调的是 SQL 分类，`prioritize`强调的是优先级，`prevent queries from starting`强调的是并发限制与排队，一句话将我个人理解的资源管理的能力都描述出来了。

下面列出市面上常见的 MPP 产品对资源管理的支持情况（Y 表示支持，N 表示不支持）：

|                                     | 资源隔离 | 隔离形式 | 分类器 / 自动路由 | 并发限制 | 排队 | 优先级 |
|:-----------------------------------:|:--------:|:--------:|:-----------------:|:--------:|:----:|:------:|
| Greenplum (Resource Groups)         | Y        | 软       | Y (User)          | Y        | Y    | Y      |
| Greenplum (Resource Queues)         | N        | 软       | Y (User)          | Y        | Y    | Y      |
| Doris (Workload Group)              | Y        | 软       | N                 | Y        | Y    |        |
| StarRocks (Resource Group)          | Y        | 软       | Y                 | Y        | N    |        |
| StarRocks (Query Queue)             | N        | 硬       | Y (Partly)        | Y        | Y    |        |
| Azure Synapse (Workload Management) | Y        | 软 / 硬  | Y                 | Y        | Y    | Y      |
| Snowflake (Multi-cluster Warehouse) | Y        | -        | N                 | Y        | Y    |        |

## 2. 它们之间用法的区别？

### 2.1 SR vs. Doris
- SR 的 Query queue 很有意思，只给用户选择限制导入或查询的 QPS，以及限制 SQL 在 BE 上最大使用率，个人认为它这样不具备资源隔离能力，只能说是一种资源保护：不至于把 BE 打满。
- 相同点：两者功能差不多，差别在名字和用法。对资源是软隔离，CPU 是以“Share”的形式设置，MEM 是以百分比的形式设置。
- 不同点：SR 支持硬限制 Big query 的资源使用，符合条件的 Big query 直接 Cancel 掉，防止误触发的 select * 把实例资源占满，Doris 暂不支持。Doris 限制并发后支持排队，SR 暂不支持。SR 支持分类器，即用户 SQL 符合某个 Pattern 则自动路由到对应资源组中，Doris 只能用户手动指定资源组。

### 2.2 Greenplum Resource Groups (GP RG)
- GP RG 仅支持将 User 指派给某个资源组，资源隔离除了 CPU 和 MEM，还支持 IO 限制的配置。
- GP RG 支持将正在运行的 Query 移动到其他资源组，不支持移动排队中的 Query。
- GP RG 是利用 Linux 的 cgroups 实现的，根据文档的描述看出是软隔离 [7]：
> When tasks in a resource group are idle and not using any CPU time, the leftover time is collected in a global pool of unused CPU cycles. Other resource groups can borrow CPU cycles from this pool. The actual amount of CPU time available to a resource group may vary, depending on the number of resource groups present on the system.

### 2.3 Greenplum Resource Queues (GP RQ)
- 与其称为 Resource Queues 不如称为 Priority Queues，一种 Query 类型对应一个 Queue。我觉得它和 SR 的查询队列有点像，其实仅仅是限制 QPS 用的，只是加上了内存限制和 Priority，而内存限制是用来保护资源的，Priority 是用来决定分配多少 CPU 的。
- 这么设计的话，RQ 就没办法做到严格的资源隔离，只能用分配的“多少”来做一定程度的资源限制，但是分配多少是优化器估计出来的，这不一定准确。

### 2.4 Azure Synapse
- Synapse 支持分类器功能，称为 Workload Classifier，可以在创建 Classifier 时指定该类 SQL 的优先级，优先级影响着后面的调度和资源分发。Synapse 的隔离形式是可配置的，支持设置最大使用资源，如果剩余量允许则可以超用。文档原文如下 [2]：
> ... additional resources can be added per request (based on resource availability).
- Synapse 同样支持排队，文档原文如下 [3]：
> Queries for the workload group that are currently queued waiting to start execution. Queries can be queue because they are waiting for resources or locks.

### 2.5 Snowflake
- 我理解 Snowflake 是以 Warehouse 来做资源隔离的，一个 Warehouse 就像是一个实例，整体的理念是：我花费力气在实例内做资源隔离，不如直接拉起新的“实例”给你用好了，只要 Resume / Suspend 做到丝滑，那么在用户体验方面对于其他产品将会是降维打击。其他产品是将 1 分成 0.3 / 0.5 / 0.2 来用，Snowflake 是分成 1 / 1 / 1 来用。这时候 Snowflake 再提出 Multi-cluster 的概念，其他产品的资源隔离模式根本没法模仿。Cluster 可以对应其他产品的 Worker 或机器，Snowflake 正是通过 Cluster 的拉起和下线实现 Warehouse 的弹性。Snowflake 对资源并发和排队的描述如下 [4]：
> If the warehouse does not have enough remaining resources to process a query, the query is queued, pending resources that become available as other running queries complete.

[1] [Workload importance - Azure Synapse Analytics](https://learn.microsoft.com/en-us/azure/synapse-analytics/sql-data-warehouse/sql-data-warehouse-workload-importance) \\
[2] [Workload isolation - Azure Synapse Analytics](https://learn.microsoft.com/en-us/azure/synapse-analytics/sql-data-warehouse/sql-data-warehouse-workload-isolation)  \\
[3] [Workload management portal monitoring - Azure Synapse Analytics](https://learn.microsoft.com/en-us/azure/synapse-analytics/sql-data-warehouse/sql-data-warehouse-workload-management-portal-monitor)  \\
[4] [Overview of Warehouses | Snowflake Documentation](https://docs.snowflake.com/en/user-guide/warehouses-overview)  \\
[5] [资源隔离 resource group StarRocks](https://docs.starrocks.io/zh/docs/2.5/administration/resource_group/)  \\
[6] [GP Managing Resources](https://docs.vmware.com/en/VMware-Greenplum/7/greenplum-database/admin_guide-wlmgmt.html)  \\
[7] [GP Using Resource Groups](https://docs.vmware.com/en/VMware-Greenplum/7/greenplum-database/admin_guide-workload_mgmt_resgroups.html)
