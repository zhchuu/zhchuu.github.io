---
layout: post
title: "Understanding CrossEntropy"
date: 2020-05-17
categories: blog
permalink: /:categories/:year/:month/:title.html
---


## 1. Entropy
我们都知道，一个事件的熵的计算如下：

$$
H(X) = -\sum_{i=1}^n p(x_i) \log(p(x_i))  \tag{1}
$$

## 2. CrossEntropy
对于神经网络解决分类问题，我们希望真实标签与输出概率的分布是一样的，不难想到用KL散度来进行衡量：

$$
\begin{aligned}
D_{KL}(p||q) &= \sum_{i=1}^n p(x_i) \log(\frac{p(x_i)}{q(x_i)})\\
&= \sum_{i=1}^n p(x_i)\log(p(x_i)) - \sum_{i=1}^n p(x_i)\log(q(x_i))  \\
&= -H(p(x))  + [-\sum_{i=1}^n p(x_i)\log(q(x_i))]
\end{aligned}  \tag{2}
$$

通常来说，$$p$$表示真实标签分布，$$q$$表示神经网络输出的概率分布。

可以看出，式子的前半部分是关于$$p$$的熵，它是固定的；所以我们只需要关注后半部分，它就是交叉熵。

所以，交叉熵为：

$$
CE = -\sum_x p(x) \log q(x)
$$

## 3. Relationship between NN Optimization and MLE
> From a perspective of probability, standard NN training via optimization is equivalent to maximum likelihood estimation (MLE) for weight.
