---
layout: post
title: "Understanding Support Vector Machines II (kernel trick)"
date: 2021-02-17
categories: blog
permalink: /:categories/:year/:month/:title.html
---

## 1. 前言

延续上一篇文章的思路（[Understanding Support Vector Machines I](https://zhchuu.github.io/blog/2021/02/understanding-support-vector-machines-i.html)），现考虑非线性的分类问题。核技巧（kernel trick）是将支持向量机运用在非线性分类问题上的关键，它不仅应用于支持向量机，还应用于其他统计学习问题。

核技巧的基本想法是通过非线性变换把输入空间映射到特征空间，使得原本在输入空间要用超曲面才能划分的样本，在特征空间中能够用超平面划分。



## 2. 核函数

核函数的定义如下：

设$$\mathcal{X}$$是输入空间，$$\mathcal{H}$$是特征空间，如果存在一个从$$\mathcal{X}$$到$$\mathcal{H}$$的映射

$$
\phi(x): \mathcal{X} \rightarrow \mathcal{H}
$$

使得对所有$$x,z \in \mathcal{X}$$，函数$$K(x, z)$$满足条件

$$
K(x, z) = \phi(x) \bullet \phi(z)
$$

则称$$K(x, z)$$为核函数，$$\phi(x)$$为映射函数。

核技巧的想法是，在学习与预测中只定义核函数$$K(x, z)$$，不需要显式地定义映射函数$$\phi$$。这样做的好处是，直接计算$$K(x, z)$$往往比较容易，如果先进行映射，再进行内积计算，反而更加复杂。还有一个有趣的事实，对于给定的核函数$$K(x,z)$$，特征空间$$\mathcal{H}$$和映射函数$$\phi$$的取法不固定，也就是说，核函数计算的结果一样，在多个维度空间中，定义的映射函数不一样。



## 3. 非线性支持向量机

观察上一篇文章（[Understanding Support Vector Machines I](https://zhchuu.github.io/blog/2021/02/understanding-support-vector-machines-i.html)）的式$$2.13$$和式$$3.8$$，最优化目标都涉及实例之间的内积（即$$x_i \bullet x_j$$），如果把它们都映射到某个能够用超平面划分样本的特征空间，那么内积运算变成$$\phi(x_i) \bullet \phi(x_j)$$，那么天然地能够使用核函数$$K(x_i, x_j)$$来代替，而不需要定义映射函数。**这正是核技巧使用的地方，也是上一篇文章中提到的，使用拉格朗日函数把原始最优化问题转换成对偶问题的好处中的第二点。**

此时对偶问题的目标函数变成

$$
W(\alpha) = \frac{1}{2} \sum_{i=1}^{N} \sum_{j=1}^{N} \alpha_i \alpha_j y_i y_j K(x_i, x_j) - \sum_{i=1}^{N} \alpha_i  \tag{3.1}
$$

在分类决策函数中的内积也可以用核函数代替，分类决策函数变成

$$
\begin{aligned}
f(x) &= sign(\sum_{i=1}^{N}\alpha_i^* y_i \phi(x_i) \bullet \phi(x) + b^*)  \\
&= sign(\sum_{i=1}^{N}\alpha_i^* y_i K(x_i, x) + b^*)
\end{aligned}  \tag{3.2}
$$



## 4. 常用核函数

函数$$K(x, z)$$满足什么条件才能成为核函数？通常说的核函数就是正定核函数（positive definite kernel function）。关于它的充分必要性证明在李航老师的《统计学习方法》有详细过程。下面列举两个常用核函数。

1. 多项式核函数（polynomial kernel function）

   $$
   K(x, z) = (x \bullet z + 1) ^p
   $$

   对应的支持向量机是一个$$p$$次多项式分类器。

2. 高斯核函数（Gaussian kernel function）

   $$
   K(x, z) = exp(-\frac{\| x - z \|^2}{2\sigma^2})
   $$

   对应的支持向量机是高斯径向基函数（radial basis function）。



## 5. 参考

《统计学习方法》, 李航
