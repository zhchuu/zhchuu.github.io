---
layout: post
title: "Understanding HyperLogLog"
date: 2022-04-17
categories: blog
permalink: /:categories/:year/:month/:title.html
---

## 1. 背景

数据库系统通常提供 Count Distinct 函数，它能够精确计算出数据的基数，基本思路是构建 Hash 表并遍历全部数据。然而当数据量太大时，Count Distinct 往往比较费时。
因此大部分数据库系统也提供 Approx Count Distinct 函数粗略计算出数据的基数，效率往往更高，如果业务上不需要精确的数值则用它更好。它们的原理来自名为 HyperLogLog 的经典算法。

基数计算算法的发展史：
- Flajolet-Martin algorithm
- LogLog
- HyperLogLog (HLL)
- HLL++
- Streaming HLL

本文由浅入深，介绍 HyperLogLog 的基本原理。


## 2. 基数估计

### 2.1 最简单的估计方法

- 假设数据分布均匀（Evenly distributed），那么对数据归一化后，用最小值除1。
  - 将数据归一化到 0 - 1 后，最小值为 $$x_{min}$$，则基数可估计为 $$1 / x_{min}$$。
  - 如果数据是复杂类型无法直接归一化，可以借助 Hash 算法。
  - 缺点：假设过于强，方差可能很大。
  
- 与上述方法本质是一样的估计：记录 Hash 后的数据的 Binary 的最大 Leading zeros。
  - 最大 Leading zeros 为 $$L_{max}$$，则基数可估计为 $$2^{L_{max}}$$。
  - 例如原数据为：$$\{0, 1, 2, 3, 4, 5, 6, 7\}$$，Hash 后得到的 Binary 值为：$$\{ 000, 001, 010, 011, 100, 101, 110, 111 \}$$，
  最多的 Leading zeros 为 000 所代表的数字，$$L_{max} = 3$$，则基数可估计为 $$2^3 = 8$$。
  - 原理和上面方法的本质是一样的，找到最多的 Leading zeros 本质上也是找最小值或最大值。
  也可以把它理解为伯努利分布，每个 Bit 实际上都是等概率的 0/1 分布，数据基数越大，让 Leading zeros 大的概率越大。
  - 缺点：同样也是假设数据分布均匀，假设太强，方差大。
  - 优点：节省内存，只需要记录 Leading zeros，32 位也仅需要 5 bits。
  
### 2.2 LogLog

- 以上计算方法最大、不可控的缺点就是方差问题，一旦有一个数字 Hash 之后的值非常小，则估计的基数将会巨大。
- 为了降低方差，可以选择的方式是用 $$m$$ 个 Hash 函数 $$\{h_1(x), h_2(x), \cdots, h_m(x)\}$$，
得到每一个最大的 Leading zeros 为 $$L_1, L_2, \cdots, L_m$$，取平均得到基数为

$$
2^{\frac{1}{m} \sum_{j=1}^m L_j}
$$

- 但是更多的 Hash 函数却带来更大的计算开销，为了降低开销又要另谋他路。Flajolet等人想到的解决办法是：
仍然用一个 Hash 函数，对于 Hash 后的值，根据 Leftmost 的 $$k$$ 位将数据分出 $$m$$ 个 Bucket，每个 Bucket 内部根据 rightmost 的 $$r$$ 位取最大 Leading zeros，
得到每一个最大的 Rightmost leading zeros 为 $$R_1, R_2, \cdots, R_m$$，最后将所有 Bucket 平均。思想类似于将数据不放回地采样，每一批样本计算一次基数，接着取平均。

- 例如输入的数据为 $$x$$：$$Hash(x) = 1011011101101100000$$，$$k = 4$$，$$m = 2^k = 16$$，$$r = 5$$，那么：将要在第 11 号 Bucket 记录最大的 Leading zeros 为5。

$$
\underbrace{1011}_{用于分 Bucket}0111011011 \underbrace{00000}_{用于取最大 Leading \ zeros}
$$

- 因此基数估计的公式为：

$$
CARDINALITY_{LogLog} = const \times m \times 2^{\frac{1}{m} \sum_{j=1}^m R_j}
$$

- 论文中推导出这样的计算方法会有 Bias，因此给出 $$const = 0.79402$$ 来调整。
- 论文同时给出：在 $$m = 2048$$，$$r = 5$$时，$$standard\_err = \frac{1.3}{\sqrt{m}} = 2.8\%$$。此时的内存使用为：$$2048 \times 5 bits = 1.2 KB$$
- $$R_1, R_2, \cdots, R_m$$本质上也是数字，存储这些数字也需要空间，5 Bits 最大能表示 $$2^{5} = 32$$，即最大能存储一个 Leading zeros 为 32 的数字，
因此能表示最大的基数为 $$2^{32}$$。这个算法之所以叫 LogLog，因为当大约知道最大的基数 $$N$$ 后，求每个 Bucket 所需空间大小的计算方法是：

$$
ceil(LogLog(N))
$$

### 2.3 HyperLogLog

- 在上述计算中，将算数平均换成调和平均能得到更低的错误率，$$standard\_err = \frac{1.04}{\sqrt{m}}$$。
- 因此基数估计的公式为：

$$
CARDINALITY_{LogLog} = const \times m \times \frac{m}{\sum_{j=1}^m (1/2^{R_j})}
$$


## Reference
[HyperLogLog in Presto: A significantly faster way to handle cardinality estimation](https://engineering.fb.com/2018/12/13/data-infrastructure/hyperloglog/) \\
[HyperLogLog in Practice: Algorithmic Engineering of a State of The Art Cardinality Estimation Algorithm](https://research.google/pubs/pub40671/)
