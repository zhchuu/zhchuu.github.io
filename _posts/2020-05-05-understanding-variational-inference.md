---
layout: post
title: "Understanding Variational Inference"
date: 2020-05-05
categories: blog
permalink: /:categories/:year/:month/:title.html
---

## 1. Preliminary

### 1.1 KL divergence
已知的知识——熵：

$$
\begin{aligned}
H = - \sum_i p(x_i)\log p(x_i)  \\
H = - \int p(x) \log p(x)dx 
\end{aligned}  \tag{1}
$$

它能衡量一个事件所包含的信息。

KL散度是衡量两个分布的距离的方法，其本质是**熵之间的比较**：

$$
\begin{aligned}
KL(p||q) &= [-\sum p(x)\log q(x)] - [-\sum p(x)\log p(x)]  \\
&= \sum p(x) \log p(x) - \sum p(x) \log q(x)  \\
&= \sum p(x) \log \frac{p(x)}{q(x)}  \\
&= - \sum p(x) \log \frac{q(x)}{p(x)}
\end{aligned}  \tag{2}
$$

注意：熵的计算中，前缀都是$$p(x)$$。因为是$$p$$与$$q$$的距离。

Two properties：

1. $$KL \geqslant 0$$

2. $$KL(p||q) \neq KL(q||p)$$

### 1.2 Conditional probability estimation

> It is a pain to get conditional prabability.

计算后验概率往往是困难的，因为根据贝叶斯公式，分母作为边缘概率是难以计算的：

$$
p(z \vert x) = \frac{p(x \vert z) p(z)}{p(x)}  \tag{3}
$$

所以后验概率一般只能被估计，有几种常见的估计方法：

1. Metropolis Hasting
    - Solution by sampling
    - More accurate
    - Takes longer to compute
    - Easier to understand
2. Variational Inference (Better approximation)
    - Deterministic solution
    - Less accurate
    - Takes less time to compute
    - Harder to understand
3. Laplace Approximation (Poor approximation)
    - Deterministic
    - Much less accurate
    - Takes little time to compute
    - Easy to understand
	
本文要介绍的是第二种：变分推导（Variational inference）

## 2. Variational inference

### 2.1 ELBO
About the relationship between $$p(x)$$ and KL divergence.

假如我们不知道后验概率分布$$p(z \vert x)$$，我们使用$$q(z)$$去估计它，$$q(z)$$是我们已知的、可控的分布。我们用KL散度来衡量估计的**质量**。

$$
KL(q(z)||p(z|x)) = -\sum q(z) \log \frac{p(z|x)}{q(z)} \tag{4}
$$

条件概率可以写成：

$$
p(z|x) = \frac{p(x, z)}{p(x)} \tag{5}
$$

将式$$(5)$$代入式$$(4)$$，得到：

$$
\begin{aligned}
KL &= - \sum q(z) \log \frac{p(x,z)}{p(x)} \frac{1}{q(z)}  \\
&= - \sum q(z) \log \frac{p(x,z)}{q(z)} \frac{1}{p(x)}  \\
&= - \sum q(z) [\log \frac{p(x,z)}{q(z)} + \log \frac{1}{p(x)}]  \\
&= - \sum q(z) [\log \frac{p(x,z)}{q(z)} - \log p(x)]  \\
&= - \sum q(z) \log \frac{p(x,z)}{q(z)} + \log p(x) \sum q(z)  \\
&= - \sum q(z) \log \frac{p(x,z)}{q(z)} + \log p(x)
\end{aligned} \tag{6}
$$

最终获得：

$$
\begin{aligned}
KL + \sum q(z) \log \frac{p(x,z)}{q(z)} &= \log p(x)  \\
KL + L &= \log p(x)
\end{aligned} \tag{7}
$$

其中，$$L$$被称为**Evidence Lower bound (ELBO)**。$$p(x)$$的值域为$$[0, 1]$$，$$\log p(x)$$的值域为$$(-\infty, 0]$$，$$KL$$为正数，所以$$L$$一定为负数。

$$L$$又可以写成：

$$
L = - KL(q(z)||p(x,z))  \tag{8}
$$

由于$$\log p(x)$$是固定的，所以为了使$$KL$$小一些，相当于使$$L$$更大，当相于使$$KL(q(z) \vert \vert p(x,z))$$变小。在解决问题时，通常选择解决$$L$$，因为条件概率是难以获得的，但是联合概率相对来说容易获得一些。

所以，为了找到一个分布$$q(z)$$逼近分布$$p(z \vert x)$$，我们优化以下式子：

$$
maximize [- KL(q(z)||p(x,z))]  \tag{9}
$$

### 2.2 Variational free energy

同样是计算$$q(z)$$和$$p(z \vert x)$$的KL散度：

$$
\begin{aligned}
KL(q(z)||p(z|x)) &= \sum q(z) \log \frac{q(z)}{p(z|x)}  \\
&= \sum q(z) \log \frac{q(z)}{p(z)p(x|z)} \\
&= KL(q(z) || p(z)) - \mathbb{E}_{q(z)}[\log p(x|z)]
\end{aligned} \tag{10}
$$

以上这个式子被称为：Variational free energy，本质上与ELBO是一样的。在论文里可能会出现这种形式的表达。

## 3. Solving problem

### 3.1 Problem

假设有两组随机变量：$$z = \{z_1, z_2, z_3\}$$，$$x = \{x_1, x_2, x_3\}$$

联合概率密度函数为：$$p(x, z) = p(x_1, x_2, x_3, z_1, z_2, z_3)$$

现在我们想找到条件概率：$$p(z \vert x) \ or \ p(z_1, z_2, z_3 \vert x_1, x_2, x_3)$$

$$
\begin{aligned}
p(z|x) &= \frac{p(x,z)}{p(x)}  \\
p(z|x) &= \frac{p(x_1, x_2, x_3, z_1, z_2, z_3)}{\int_{z_1} \int_{z_2} \int_{z_3} p(x_1, x_2, x_3, z_1, z_2, z_3) dz_1 dz_2 dz_3}
\end{aligned}  \tag{11}
$$

> 解决方法是：用$$q(z_1, z_2, z_3)$$去**估计**$$p(z_1, z_2, z_3 \vert x_1, x_2, x_3)$$。

利用式$$(7)$$，可得：

$$
\begin{aligned}
L &= \sum_z q(z) \ln \frac{p(x,z)}{q(z)}  \\
L &= \sum_{z_1} \sum_{z_2} \sum_{z_3} q(z_1, z_2, z_3) \ln \frac{p(x_1, x_2, x_3, z_1, z_2, z_3)}{q(z_1, z_2, z_3)}
\end{aligned}  \tag{12}
$$

> 我们的目标是找到一个$$q(z_1, z_2, z_3)$$最大化$$L$$。

不幸的是，这个$$q(z_1, z_2, z_3)$$好难找呀，即使是VI的发明者，当时也没有一个很好的想法去解决这个问题。

### 3.2 Mean-Field VI

**我们假设$$z_1, z_2, z_3$$是相互独立的**，那么：

$$
q(z_1, z_2, z_3) = q(z_1)q(z_2)q(z_3) = \prod_i q(z_i)  \tag{13}
$$

将式$$(13)$$代入式$$(12)$$：

$$
\begin{aligned}
L &= \sum_{z_1} \sum_{z_2} \sum_{z_3} q(z_1)q(z_2)q(z_3) \ln \frac{p(x,z)}{q(z_1)q(z_2)q(z_3)}  \\
L &= \sum_{z_1} \sum_{z_2} \sum_{z_3} q(z_1)q(z_2)q(z_3) [\ln p(x,z) - \ln [q(z_1)q(z_2)q(z_3)]] \\
L &= \sum_{z_1} \sum_{z_2} \sum_{z_3} q(z_1)q(z_2)q(z_3) [\ln p(x,z) - \ln q(z_1) - \ln q(z_2) - \ln q(z_3)]
\end{aligned}  \tag{14}
$$

> 我们可以转变为分别解决$$q(z_1)$$，$$q(z_2)$$，$$q(z_3)$$。

> 假设我们已知$$q(z_2)$$和$$q(z_3)$$，我们要怎么求$$q(z_1)$$？

式$$(14)$$可写成：

$$
L = \sum_{z_1} \sum_{z_2} \sum_{z_3} q(z_1) q(z_2) q(z_3) [\ln p(x,z) - \ln q(z_1) - [\ln q(z_2) + \ln q(z_3)]]
$$

我们将它稍微展开可以获得三个部分：

$$
\sum_{z_1} \sum_{z_2} \sum_{z_3} q(z_1) q(z_2) q(z_3) \ln p(x,z) \tag{14.1}
$$

$$
- \sum_{z_1} \sum_{z_2} \sum_{z_3} q(z_1) q(z_2) q(z_3) \ln q(z_1)  \tag{14.2}
$$

$$
- \sum_{z_1} \sum_{z_2} \sum_{z_3} q(z_1) q(z_2) q(z_3) [\ln q(z_2) + \ln q(z_3)]  \tag{14.3}
$$

我们对以上三个部分一个一个来看：
对于$$(14.3)$$：把$$q(z_1)$$单独提出来

$$
- \sum_{z_1} q(z_1) \sum_{z_2} \sum_{z_3} q(z_2) q(z_3) [\ln q(z_2) + \ln q(z_3)]
$$

显然，后半部分为常数（因为我们已知$$q(z_1)$$和$$q(z_2)$$），即：

$$
part3 = - \sum_{z_1} q(z_1) * K
$$

对于$$(14.2)$$：把$$q(z_1)$$单独提出来

$$
- \sum_{z_1} q(z_1) \ln q(z_1) \sum_{z_2} \sum_{z_3} q(z_2) q(z_3) 
$$

显然，后半部分是概率求和，结果为1，即：

$$
part2 = - \sum_{z_1} q(z_1) \ln q(z_1)
$$

对于$$(14.1)$$：把$$q(z_1)$$单独提出来

$$
\sum_{z_1} q(z_1) \sum_{z_2} \sum_{z_3}  q(z_2) q(z_3) \ln p(x_1, x_2, x_3, z_1, z_2, z_3)
$$

此时回想期望的计算公式：

$$
\begin{aligned}
E[x] &= \sum x p(x)  \\
E[f(x)] &= \sum f(x) p(x)
\end{aligned}
$$

其中，$$f(x)$$是某个函数，$$p(x)$$是概率，形式与$$(14.1)$$一样，所以：

$$
part1 = \sum_{z_1} q(z_1) E_{z_2, z_3}[\ln p(x_1, x_2, x_3, z_1, z_2, z_3)]
$$

此时把上面三个部分合在一起：

$$
\begin{aligned}
L &= \sum_{z_1} q(z_1) E_{z_2, z_3}[\ln p(x, z)] - \sum_{z_1} q(z_1) \ln q(z_1) - \sum_{z_1} q(z_1) * K  \\
L &= \sum_{z_1} q(z_1) [E_{z_2, z_3}[\ln p(x, z)] - K] - \sum_{z_1} q(z_1) \ln q(z_1)
\end{aligned}  \tag{15}
$$

**Further more**

通过大量简单的数学运算，式$$(14)$$已经被大量简化，成为式$$(15)$$，再把式$$(15)$$写成：

$$
\begin{aligned}
L = \sum_{z_1} q(z_1) &[E_{z_2, z_3}[\ln p(x, z)] - K_1 - K_2] \\
 &- \sum_{z_1} q(z_1) \ln q(z_1)
\end{aligned}  \tag{16}
$$

我们接下来要证明：

$$
\begin{aligned}
ln f(x, z) &= E_{z_2, z_3}[\ln p(x, z)] - K_1  \\
f(x, z) &= exp(-K_1)*exp(E_{z_2, z_3}[\ln p(x, z)])  \\
f(x, z) &= C*exp(E_{z_2, z_3}[\ln p(x, z)])
\end{aligned}  \tag{17}
$$

将式$$(17)$$代入式$$(16)$$：

$$
\begin{aligned}
L &= \sum_{z_1} q(z_1) [\ln f(x,z) - K_2] - \sum_{z_1} q(z_1) \ln q(z_1) \\
L &= \sum_{z_1} q(z_1) \ln f(x,z) - \sum_{z_1} q(z_1) \ln q(z_1) - K_2*\sum_{z_1}q(z_1) \\
L &= \sum_{z_1} q(z_1) \ln \frac{f(x,z)}{q(z_1)} + constant
\end{aligned}  \tag{18}
$$

其中：$$\sum_{z_1} q(z_1) \ln \frac{f(x,z)}{q(z_1)} = - KL(q(z_1) \vert \vert f(x,z))$$

> 所以，为了最大化$$L$$，我们需要最小化$$KL(q(z_1) \vert \vert f(x,z))$$。

> 那么$$f(x,z)$$是什么？

回到式$$(17)$$：

$$
E_{z_2, z_3}[\ln p(x, z)] = \sum_{z_2} \sum_{z_3} q(z_2) q(z_3)lnp(x,z)
$$

所以：

$$
\begin{aligned}
f(x,z) &= q(z_1) = C_1*exp(\sum_{z_2} \sum_{z_3} q(z_2)   q(z_3) \ln p(x,z))  \\
f(x,z) &= q(z_2) = C_2*exp(\sum_{z_1} \sum_{z_3} q(z_1)   q(z_3) \ln p(x,z))  \\
f(x,z) &= q(z_3) = C_3*exp(\sum_{z_1} \sum_{z_2} q(z_1)   q(z_2) \ln p(x,z))  \\
\end{aligned}  \tag{19}
$$

现在问题就很令人困扰，只需要得到$$q(z_1)$$，$$q(z_2)$$，$$q(z_3)$$就可以计算出$$q(z)$$了，但是分别求$$q(z_i)$$又求不出来，看似离答案很近，实则很远。

**解决**

解决方案是，在计算$$q(z_1)$$时，随机选择$$q(z_2)$$和$$q(z_3)$$；得到$$q(z_1)$$后，计算$$q(z_2)$$时候再随机选择$$q(z_3)$$；计算$$q(z_3)$$时，利用已经计算出的$$q(z_1)$$和$$q(z_2)$$。如此循环，直到收敛。

以上是Mean-Field VI的粗略推导过程，可能不是特别严谨和范化，想进一步了解推导过程的话可以点击下面连接。

## 参考
[Variational Inference](https://www.youtube.com/watch?v=uKxtmkfeuxg)
