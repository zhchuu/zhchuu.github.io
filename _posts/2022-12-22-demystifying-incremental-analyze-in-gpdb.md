---
layout: post
title: "Demystifying Incremental ANALYZE in Greenplum Database"
date: 2022-12-22
categories: blog
permalink: /:categories/:year/:month/:title.html
---

## 0. Prelinminary

从业务的角度看，以时间（年月日）分区是较为常见的，一般不会对过去时间的分区进行 DDL 或 DML，只会查询旧分区和插入数据到新分区。
因此每次新建分区就对父表 Analyze 是比较繁重的事情，因为大部分数据都没有变化。为了优化该问题，Greenplum Database（GPDB）提出将子分区表的统计信息合并到父表上（详见参考），这样每次新建分区只需要 Analyze 新分区即可。我把它称为 Incremental analyze（非官方、非正式表达）。

我在官网没有找到对这个过程的详细介绍，说明它对用户是无感的。本文将深入介绍该过程，需要读者有一定的 PostgreSQL（PG） 的统计信息、Analyze 过程等前置知识，可以先阅读前面一篇文章[[A Recap on ANALYZE in PostgreSQL](https://zhchuu.github.io/blog/2022/11/a-recap-on-analyze-in-postgresql.html)]。

## 1. How to Merge Statistics

阅读代码之前，先看看各种统计信息用什么算法合并，对于表的任意一列，假设子表统计信息都完整：

- 基本统计信息：Rows / Width / Null-fraction
  - Rows：直接将子表的行数相加即可得到总行数。
  - Width：已知子表的平均宽度和行数，根据比例可以轻松得到父表的平均宽度。
  - Null-fraction：同上。

- Number of Distinct Value（NDV）
  - 借助 HLL 算法（在之前的 Post 中有介绍[[Understanding HyperLogLog](https://zhchuu.github.io/blog/2022/04/understanding-hyperloglog.html)]）。
  - 把子表当作中间状态，父表当作最终状态，合并就不成问题。那么只需要考虑中间状态的存储位置，GPDB 选择存在 pg_statistic 系统表中。

- Most Common Value（MCV）
  - pg_statistic 中不仅记录 MCV 的数值（stanumbersN），还记录了频率（stavaluesN），依靠它们得到父表的 MCV 不是问题。

- Histogram
  - 合并直方图也是可行的，因为 pg_statistic 记录直方图有一个性质：尽可能保证每根柱子同高。
  - 因此 stanumbersN 的数值相当于把表的数据平均分，可以通过适当的算法进行合并得到父表直方图（后文介绍）。

了解合并统计信息是可行的后，下面带着几点疑问去阅读代码：
1. 统计信息的合并怎么嵌入到原来的 Analyze 的流程中，实现无感。
2. 子表的 HLL 信息存在哪，怎么存。
3. 合并 MCV 的过程。
4. 合并 Histogram 的过程。


## 2. Procedure

### 2.1 Incremental Analyze 流程的嵌入


```C
// src/backend/commands/analyze.c
if (get_rel_relkind(attr->attrelid) == RELKIND_PARTITIONED_TABLE &&
    !get_rel_relispartition(attr->attrelid) &&
    leaf_parts_analyzed(stats->attr->attrelid, InvalidOid, va_cols, stats->elevel) &&
    ((!OidIsValid(eqopr)) || op_hashjoinable(eqopr, stats->attrtypid)))
{
    stats->merge_stats = true;
    stats->compute_stats = merge_leaf_stats;
    stats->minrows = 300 * attr->attstattarget;
}
```

- 在 ```VacAttrStats``` 结构体中增加变量 ```merge_stats``` 来标识是否达到合并的条件。
- 如果达到了，则使用 ```merge_leaf_stats``` 函数来计算统计信息，这样一来所有合并的逻辑都放到函数里即可，不会破坏原先的 Analyze 流程。
- 是否达到合并条件：
  - 具备分区属性。
  - 不是子表而是父表。
  - 统计信息合格（没有缺漏），上文中的 ```leaf_parts_analyzed``` 函数。
  - 该列的类型可哈希，做法是从 pg_operator 表中拿 oprcanhash 字段来判断，暂不支持哈希的类型（如 box / path / line / point）会在 NDV 合并的时候出现问题，因此这里拦住。
  
### 2.2 HLL 中间信息存储

```C
// src/include/catalog/pg_statistic.h
#define STATISTIC_NUM_SLOTS  5

// src/backend/commands/analyze.c
if (stakind > 0)
{
    stats->stakind[STATISTIC_NUM_SLOTS-1] = stakind;
    stats->stavalues[STATISTIC_NUM_SLOTS-1] = hll_values;
    stats->numvalues[STATISTIC_NUM_SLOTS-1] =  1;
    stats->statyplen[STATISTIC_NUM_SLOTS-1] = hll_length;
}
```

- 前面说到，GPDB 把 HLL 存储在 pg_statistic 中，稍微了解 PG 的应该都知道：pg_statistics 系统表中有 5 个 Slot，即 stakind1, stakind2, ..., stakind5。
其中 HLL 被序列化成 Byte array 存储在第五个 Slot。

### 2.3 合并 MCV

```C
// src/backend/commands/analyzeutils.c
MCVFreqPair **
aggregate_leaf_partition_MCVs(...)
```
- 整体思路是利用子表的 MCV 信息和 Row count 信息计算出子表各个 MCV 合并后的数量和比例，如果小于一个阈值则去掉，认为合并后看起来不算 Common 了。
- 此处会记录两个值：```num_mcv``` 和 ```rem_mcv```，前者为合并后仍然是 Common 的数量，后者为不那么 Common 的数量。
- 后者的意义是为了弥补信息损失，这些不那么 Common 的值不应该直接丢掉，而是放到 Histogram 中展示。


### 2.4 合并 Histogram

```C
// src/backend/commands/analyzeutils.c
int
aggregate_leaf_partition_histograms(...)
```
- 以如下例子来说明合并的过程：
  - 假如有三个子表的直方图为：
```
hist(prt_1): {0,19,38,59}
hist(prt_2): {2,18,40,62}
hist(prt_3): {1,22,39,61}
```
  - 并且它们的数据量为：
```
nTuples(prt_1) = 300
nTuples(prt_2) = 270
nTuples(prt_3) = 330
```
  - 因此每个柱子的面积是：
```
bucketSize(prt_1) = 300/3 = 100
bucketSize(prt_2) = 270/3 = 90
bucketSize(prt_3) = 330/3 = 110
```
  - 目标是合并出直方图形如：```hist(agg): {A,B,C,D}```，表示 ```A-B```，```B-C```，```C-D``` 之间分别有 ```(300 + 270 + 330) / 3 = 300``` 条数据。
  因此定义 ```bucketSize(agg) = 300```，表示合并后直方图“一根柱子”的面积是 300。
  - 我们要做的是逐步取子表直方图的小柱子，合并成父表直方图的大柱子。
  - 维护一个优先队列，里面放着 3 个（子表数量）值，分别表示目前三个子表的最小边界，一开始是```{0,2,1}```，取第一个边界```min{0,2,1} = 0```，接着把 19 放进去，始终保持优先队列里有 3 个值。
  - 接下来取第二个边界，逐渐从优先队列中拿最小值，要求面积填满 300。直到队列里为```{19,18,22}```，此时拿```min{19,18,22} = 18```（放入 40），因为知道 ```2-18``` 有 90 的面积，但还不够占满 300，于是继续拿。
  - 直到拿走 19 和 22 后，面积求和为 ```90 + 100 + 110 = 300```，那么第二个边界出现了，即为 22，此时直方图为```{0,22}```。
  - 以此类推，最终直方图为```{0,22,40,62}```。


## 3. Summary

以上就是 GPDB 利用子表统计信息合并出父表统计信息的核心过程，没有什么难以理解的部分。
我没有复制粘贴太多代码，只是将函数名拷贝出来方便读者去源码中搜索。
为了方便读者理解，我也省略了许多实现细节，实际实现上 GPDB 还是用了不少很棒的技巧，读者可以去 Reference 的链接中阅读。

有了这个特性，新增子表后，获得父表统计信息不需要重新 Analyze 整张父表，只需要 Analyze 新增的子表即可。个人测试了一下能有不少的速度提升，分区表越多提升越明显。


## Reference

[[Utilize hyperloglog and merge utilities to derive root table statistics](https://github.com/greenplum-db/gpdb/commit/9c1b1ae3afe999031f2e43a87f7dbae61b9c7fbe)]
