---
layout: post
title: "A Recap on ANALYZE in PostgreSQL"
date: 2022-11-17
categories: blog
permalink: /:categories/:year/:month/:title.html
---

## 0. Preliminary
Analyze 收集数据库中表内容的统计信息，为引擎生成良好的执行计划提供有力的支持。本文总结 PostgresSQL（PG）的 Analyze 整体流程。


## 1. PG Analyze 调用链路
- 编译运行打印堆栈

从 GitHub 上拉取 PG 代码，编译运行，执行 Analyze 并用 GDB 打断点得到如下堆栈：
```c
#0  in do_analyze_rel at analyze.c:1059
#1  in analyze_rel at analyze.c:262
#2  in vacuum at vacuum.c:492
#3  in ExecVacuum at vacuum.c:275
#4  in standard_ProcessUtility at utility.c:866
#5  in PortalRunUtility at pquery.c:1158
#6  in PortalRunMulti at pquery.c:1315
#7  in PortalRun at pquery.c:791
#8  in exec_simple_query at postgres.c:1243
#9  in PostgresMain at postgres.c:4505
#10 in BackendRun at postmaster.c:4491
#11 in BackendStartup at postmaster.c:4219
#12 in ServerLoop at postmaster.c:1809
#13 in PostmasterMain at postmaster.c:1481
#14 in main at main.c:19
```

- Analyze 作为普通 SQL 进入到 ``exec_simple_query`` 函数中，并且它属于 Utility 语句，因此走到 ``standard_ProcessUtility`` 函数。

- 由于 PG 遗留的历史原因，Vacuum 和 Analyze 都会走到 ``ExecVacuum`` 函数中，原因是 Vacuum 通常和 Analyze 顺序执行。

- 进入 ``vacuum`` 函数后，决定做 Vacuum 还是 Analyze 由 VacuumParams 结构体中的 options 变量决定，变量“定义域”如下，用 2 的次方来标记每个元素方便进行与或非逻辑运算。
```c
// src/include/commands/vacuum.h
/* flag bits for VacuumParams->options */
#define VACOPT_VACUUM 0x01		/* do VACUUM */
#define VACOPT_ANALYZE 0x02		/* do ANALYZE */
#define VACOPT_VERBOSE 0x04		/* output INFO instrumentation messages */
#define VACOPT_FREEZE 0x08		/* FREEZE option */
#define VACOPT_FULL 0x10		/* FULL (non-concurrent) vacuum */
#define VACOPT_SKIP_LOCKED 0x20 /* skip if cannot get lock */
#define VACOPT_PROCESS_TOAST 0x40	/* process the TOAST table, if any */
#define VACOPT_DISABLE_PAGE_SKIPPING 0x80	/* don't skip any pages */
```

- 函数 do_analyze_rel 才是真正干活的地方。

## 2. do_analyze_rel

根据个人理解，将该函数做的内容拆分成以下几个步骤：

1. 创建名为 anl_context 的 MemoryContext，在该 Context 下保证内存安全；获取表的 Owner 并切换过去，在表 Owner 的权限下执行操作。

2. 函数参数 va_cols 表示要 Analyze 的列，如果用户没有指定则该参数为空，隐式表示所有列；对要 Analyze 的列执行 ``examine_attribute`` 函数，它的主要工作是：

   2.1 为列初始化（内存分配，变量填写等等）VacAttrStats 结构体，这个结构体存储统计信息；

   2.2 确定该列要采样出多少样本来提供统计信息；

   2.3 根据该列的类型属性，确定用哪一个函数来计算统计信息：
       
       a. 如果该列支持“=”和“<=”比较，那么它是 Scalar 类型，用 compute_scalar_stats 函数；
       b. 如果该列仅支持“=”比较，用 compute_distinct_stats 函数；
       c. 否则，用 compute_trivial_stats 函数。
       
   三个函数的区别在于获取统计信息的丰富程度，见文末尾表格。

3. 根据第 2 步的采样目标行数（记为 targrows）采样出所需要的样本，采样方法如下（翻译自 ``acquire_sample_rows`` 函数的注释）：

    a. 阶段一：选择至多 targrows 个随机的 Blocks；

    b. 阶段二：对于每个 Block 采用 Vitter 算法采样。
    
4. 对于每一列，创建名为 col_context 的 MemoryContext，使用第 2 步的计算函数对第 3 步得到的样本计算统计信息。

5. 将统计信息存入 pg_statistic / pg_class 系统表中。

|               | compute_scalar_stats | compute_distinct_stats | compute_trivial_stats |
|:-------------:|:--------------------:|:----------------------:|:---------------------:|
| Width         | Yes                  | Yes                    | Yes                   |
| Null-fraction | Yes                  | Yes                    | Yes                   |
| NDV           | Yes                  | Yes                    |                       |
| MCV           | Yes                  | Yes                    |                       |
| Histogram     | Yes                  |                        |                       |
