---
layout: post
title: "Modern NFs algorithms"
date: 2021-01-08
categories: blog
permalink: /:categories/:year/:month/:title.html
---

# 前言

粗略看了一些更复杂的Normalizing flows的论文，非常羞愧地表示能完全理解的真的少。这里先记录一些容易理解的方法。我一般只对构建Flows的部分感兴趣，具体用于什么场景不太在意。

要初步了解Normalizing flows可移步：[Understanding Normalizing Flows](https://zhchuu.github.io/blog/2020/04/understanding-normalizing-flows.html)


# NADE
论文：[Neural Autoregressive Distribution Estimation](https://arxiv.org/abs/1605.02226) (JMLR 2016)

主要贡献：提出了一种自回归的密度估计方法，并用神经网络进行密度估计。

假设有$$D$$维数据$$\vec x = \{x_1, x_2, \cdots, x_d\}$$，它的联合概率密度为：

$$
q(x) = q_1(x_1)*q_2(x_2|x_1)*\cdots*q_d(x_d|x_{<d}) \tag{1}
$$

为每一个条件概率或边缘概率选择一个maps，组成triangular maps。


# Planar flows
论文：[Variational Inference with Normalizing Flows](https://arxiv.org/abs/1505.05770)

主要贡献：提出使用NFs来帮助VI估计后验概率。文章中关于VI的部分不详细解释，主要介绍它提出来的Invertible Linear-time Transformations。

$$
f(z) = z + uh(w^{T}z+b) \tag{2}
$$

其中，$$w \in \mathbb{R}^D$$，$$u \in \mathbb{R}^D$$，$$b \in \mathbb{R}$$，$$h(\cdot)$$是一个平滑的**element-wise**的、非线性函数，其导数为$$h'(\cdot)$$。

> 由于函数$$h(\cdot)$$是element-wise的，所以雅可比矩阵是一个对角矩阵，行列式的计算直接是对角线上元素相乘。（我的理解）

雅可比行列式表示为：

$$
\psi(z) = h'(w^Tz+b)w  \\
|det \frac{\partial f}{\partial z}| = |det(I+u\psi(z)^T)| = |1+u^T\psi(z)|
$$


# RealNVP
论文：[Density Estimation Using Real NVP](https://arxiv.org/abs/1605.08803) (ICLR 2017)

主要贡献：提出Real-valued Non-Volumn Preserving (Real NVP)的映射方式，这是一系列的强大的、稳定可逆的和可学习的映射。本文应用场景是非监督学习，希望对未标记的数据使用生成概率模型进行建模。

假设数据为$$X = \{x_1, x_2, x_3, \cdots, x_D\}$$，映射到空间$$Y$$。显然，数据维度为$$D$$。

它建立的映射是这样的：

$$
\begin{aligned}
y_{1:d} &= x_{1:d} \\
y_{d+1:D} &= x_{d+1:D} \odot exp(s(x_{1:d})) + t(x_{1:d})
\end{aligned} \tag{3}
$$

$$
\begin{aligned}
x_{1:d} &= y_{1:d} \\
x_{d+1:D} &= (y_{d+1:D} - t(y_{1:d})) \odot exp(-s(y_{1:d}))
\end{aligned} \tag{4}
$$

其中，$$s$$和$$t$$代表$$scale$$和$$translation$$，是$$R^d \mapsto R^{D-d}$$的函数，$$\odot$$为element-wise product。
直接分析公式：前$$d$$维的值不作任何变换；$$d+1$$到$$D$$维的值与$$x_{1:d}$$有关。

直观地看：

$$
\begin{aligned}
y_1 &= x_1 \\
y_2 &= x_2 \\
& \vdots \\
y_d &= x_d \\
y_{d+1} &= T_{d+1}(x_1, x_2, \cdots, x_d, x_{d+1}) \\
y_{d+2} &= T_{d+2}(x_1, x_2 \cdots, x_d, x_{d+2}) \\
& \vdots \\
y_D &= T_D(x_1, x_2, \cdots, x_d, x_D)
\end{aligned}  \tag{5}
$$

此时可以直白地看出，它的雅可比矩阵为：

$$
\frac{\partial y}{\partial x^T} =
\left[
    \begin{matrix}
        \mathbb{I}_d & 0 \\
        \frac{\partial y_{d+1:D}}{\partial x_{1:d}^T} & diag(exp[s(x_{1:d})])
    \end{matrix}
\right] \tag{6}
$$

其中，$$diag(exp[s(x_{1:d})])$$为对角矩阵，并且它与$$x_{d+1:D}$$无关，所以完全无需考虑$$s$$和$$t$$的偏导，可以增加复杂度，这里能用神经网络。

再直观地将雅可比矩阵表示出来：

$$
\left[
\begin{matrix}
\frac{\partial T_1}{\partial x_1} & 0 & \cdots & \cdots & \cdots & \cdots & 0 \\
0 & \frac{\partial T_2}{\partial x_2} & \cdots & \cdots & \cdots & \cdots & 0 \\
\vdots & \vdots & \ddots & \ddots & \ddots & \cdots & \vdots \\
0 & 0 & \cdots & \frac{\partial T_d}{\partial x_d} & \cdots & \cdots  & 0 \\
\frac{\partial T_{d+1}}{\partial x_1} & \frac{\partial T_{d+1}}{\partial x_2} & \cdots & \frac{\partial T_{d+1}}{\partial x_d} & \frac{\partial T_{d+1}}{\partial x_{d+1}} & \cdots & 0 \\
\vdots & \vdots & \ddots & \ddots & \ddots & \ddots & 0 \\
\frac{\partial T_{D}}{\partial x_1} & \frac{\partial T_{D}}{\partial x_2} & \cdots & \frac{\partial T_{D}}{\partial x_d} & 0 & \cdots & \frac{\partial T_D}{\partial x_D} \\
\end{matrix}
\right] \tag{7}
$$

这就非常直观地看出这是一个三角矩阵。

由于$$1:d$$维的值维持不变，为了解决这个问题，在进行多层叠加的时候，让上一层没变的改变，上一层改变的不变。
