---
layout: post
title: "Taylor Expansion"
date: 2020-05-21
categories: blog
permalink: /:categories/:year/:month/:title.html
---


## 1. Basic

泰勒展开的用处：用一个**多项式**去拟合任意一个函数。

泰勒展开的基本思想：如何评判两个函数是否相近？

> 1. 在某一点的取值一样；
> 2. 在该点的一阶导数取值一样；
> 3. 在该点的二阶导数取值一样；\\
> $$\vdots$$
> 4. 在该点的$$n$$阶导取值一样；

假如我们要拟合的函数为$$f(x)$$，值域为$$(-\infty, \infty)$$，且处处连续可导。为了找到另外一个函数$$g(x)$$去拟合它，必须要求这个函数同样处处可导，而且能取得$$n$$阶导数。最简单的一个函数即为：**$$n$$阶多项式**。

## 2. Induction

在$$f(x)$$上任意选一点，假设为$$(0, f(0))$$，构造函数$$g(x)$$为：

$$
g(x) = a_0 + a_1x + a_2 x^2 + \cdots + a_n x^n \tag{1}
$$

我们要获得一组参数，$$a_0, a_1, a_2, \cdots, a_n$$，使得$$g(x)\approx f(x)$$。现在逐一满足上面提到的条件，首先是**在该点取值一样**：

$$
g(0) = f(0) = a_0 \tag{2}
$$

**在该点的$$n$$阶导数取值一样**：

$$
g^{n}(0) = f^{n}(0) = a_n \times n! \tag{3}
$$

得出：

$$
a_n = \frac{f^{n}(0)}{n!} \tag{4}
$$

所以：

$$
g(x) = f(0) + \frac{f^{1}(0)}{1!}x + \frac{f^{2}(0)}{2!}x^2 + \cdots + \frac{f^{n}(0)}{n!}x^n \tag{5}
$$

当初始选择的点不是0时，例如选择$$(x_0, f(x_0))$$，构造的$$g(x)$$为：

$$
g(x) = a_0 + a_1(x-x_0) + a_2 (x-x_0)^2 + \cdots + a_n (x-x_0)^n \tag{6}
$$

经过相同的推导：

$$
g(x) = f(x_0) + \frac{f^{1}(x_0)}{1!}(x-x_0) + \frac{f^{2}(x_0)}{2!}(x-x_0)^2 + \cdots + \frac{f^{n}(x_0)}{n!}(x-x_0)^n \tag{7}
$$

式$$(7)$$就是泰勒展开式，写成：

$$
g(x) \approx f(x)
$$

如果要把约等于变成等于，要把$$g(x)$$最后一项去掉，换成省略号。

## 3. Further
扩展到二阶函数的泰勒展开，函数$$f(x_1, x_2)$$在$$x_0(x_{10}, x_{20})$$点处的泰勒展开式为：

$$
f(x_1, x_2) = f(x_{10}, x_{20}) + f_{x_1}(x_{10}, x_{20})\Delta x_1 + f_{x_2}(x_{10}, x_{20})\Delta x_2 + \\
\frac{1}{2}[f_{x_1 x_1}(x_{10}, x_{20})\Delta x_1^2 + 2f_{x_1 x_2}(x_{10}, x_{20})\Delta x_1 \Delta x_2 + f_{x_2 x_2}(x_{10}, x_{20})\Delta x_2^2] + \cdots  \tag{8}
$$

其中，$$\Delta x_1 = x_1 - x_{10}$$，$$\Delta x_2 = x_2 - x_{20}$$，$$f_{x_1} = \frac{\partial f}{\partial x_1}$$，$$f_{x_2} = \frac{\partial f}{\partial x_2}$$，$$f_{x_1 x_1} = \frac{\partial^2 f}{\partial^2 x_1}$$，$$f_{x_2 x_2} = \frac{\partial^2 f}{\partial^2 x_2}$$，$$f_{x_1 x_2} = \frac{\partial^2 f}{\partial x_1 \partial x_2}$$。

将上述展开式写成矩阵形式，则有：

$$
f(x) = f(x_0) + \bigtriangledown f(x_0)^{\mathrm T} \Delta x + \frac{1}{2} \Delta x^{\mathrm T} G(x_0) \Delta x + \cdots  \tag{9}
$$

其中，$$\Delta x = \begin{bmatrix} \Delta x_1 \\ \Delta x_2\end{bmatrix}$$，$$\bigtriangledown f(x_0) = \begin{bmatrix} \frac{\partial f}{\partial x_1} \\ \frac{\partial f}{\partial x_2} \end{bmatrix}$$是函数$$f(x_1, x_2)$$在$$x_0$$的梯度，矩阵

$$
G(x_0) = \begin{bmatrix} \frac{\partial^2 f}{\partial x_1^2} & \frac{\partial^2 f}{\partial x_1 \partial x_2} \\
\frac{\partial^2 f}{\partial x_2 \partial x_1} & \frac{\partial^2 f}{\partial x_2^2}
\end{bmatrix}_{x_0}  \tag{10}
$$

是函数$$f(x_1, x_2)$$在$$x_0$$点的$$2 \times 2$$黑森矩阵。它是由函数$$f(x_1, x_2)$$在$$x_0$$点的所有二阶偏导数组成的方阵。黑森矩阵为对称矩阵。

## 4. Further more
将二元函数的泰特展开推广到多元函数，函数$$f(x_1, x_2, \cdots, x_n)$$在$$x_0(x_1, x_2, \cdots, x_n)$$点处的泰勒展开式为：

$$
f(x) = f(x_0) + \bigtriangledown f(x_0)^{\mathrm T} \Delta x + \frac{1}{2} \Delta x^{\mathrm T} G(x_0) \Delta x + \cdots  \tag{11}
$$

其中

$$
\bigtriangledown f(x_0) = \begin{bmatrix} \frac{\partial f}{\partial x_1} & \frac{\partial f}{\partial x_2} & \cdots  & \frac{\partial f}{\partial x_n}\end{bmatrix}  \tag{12}
$$

为函数$$f(x)$$在$$x_0(x_1, x_2, \cdots, x_n)$$处的梯度，

$$
G(x_0) = \begin{bmatrix} \frac{\partial^2 f}{\partial x_1^2} & \frac{\partial^2 f}{\partial x_1 \partial x_2} & \cdots &  \frac{\partial^2 f}{\partial x_1 \partial x_n} \\
\frac{\partial^2 f}{\partial x_2 \partial x_1} & \frac{\partial^2 f}{\partial x_2^2} & \cdots & \frac{\partial^2 f}{\partial x_2 \partial x_n} \\
\vdots & \vdots & \ddots & \cdots \\
\frac{\partial^2 f}{\partial x_n \partial x_1} & \frac{\partial^2 f}{\partial x_n \partial x_2} & \cdots & \frac{\partial^2 f}{\partial x_n^2}
\end{bmatrix}  \tag{13}
$$

为函数$$f(x)$$在$$x_0$$点的$$n \times n$$黑森矩阵。若函数有$$n$$次连续性，则黑森矩阵是对称矩阵。
