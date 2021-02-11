---
layout: post
title: "Understanding the Fast InvSqrt Algorithm"
date: 2021-02-11
categories: blog
permalink: /:categories/:year/:month/:title.html
---

## 1. 前言

计算机想要计算一个浮点数的平方根，一般的做法是不断迭代使得估计值靠近真实值，最简单的迭代方法是二分查找，[LeetCode 69.](https://leetcode.com/problems/sqrtx/)题就可以这么做出来。但毕竟二分查找的复杂度是$$\log n$$，在一些极限场景下可能还是太慢，有没有更快的方法呢？那么就要利用一些数值分析和浮点数表示的知识，本文就来分析著名的快速计算浮点数反平方根算法（也称为卡马克算法），该算法是由**John Carmack（约翰 · 卡马克）**在1999年开发一款第一人称射击游戏时写出来的，直到2002～2003年人们在阅读该游戏源码时，其惊人的效率和叹为观止的解法震惊众人，该算法才出现在大众视野，被誉为经典。

反平方根算法要求输入任意一个浮点数$$a$$（在当时是32位浮点数），计算$$\frac{1}{\sqrt{a}}$$。


## 2. 预备知识

### 2.1 牛顿法

牛顿法是机器学习中用得比较多的一种优化算法，基本思想是将函数在点$$x_k$$处二阶泰勒展开，用一阶导数（梯度）和二阶导数（Hessian矩阵）对目标函数进行近似，把近似后的函数的极小点作为新的迭代点，如此迭代直到得到精度足够的估计值。

假设目标函数为$$f(x)$$，在$$x_k$$处进行二阶泰勒展开（不熟悉泰勒展开的可看[这篇](https://zhchuu.github.io/blog/2020/05/taylor-expansion.html)）：

$$
f(x) \approx f(x_k) + f'(x_k)(x-x_k) + \frac{1}{2} f''(x_k)(x-x_k)^2  \tag{2.1}
$$

极值点在$$f'(x) = 0$$时取得，所以对上式求导并令其为0：

$$
f'(x) \approx f'(x_k) + f''(x_k)(x-x_k) = 0  \tag{2.2}
$$

得到：

$$
x = x_k - \frac{f'(x_k)}{f''(x_k)}  \tag{2.3}
$$

以上就是牛顿法的更新公式，利用函数在点$$x_k$$处的一阶导数和二阶导数进行更新，在其他的教程中可能会把$$f'(x_k)$$写为$$g_k$$以表示梯度，把$$f''(x_k)$$写为$$H_k$$以表示Hessian矩阵，殊途同归。

### 2.2 IEEE 754浮点数表示

关于IEEE 754标准的浮点数表示的建立历史可以移步到[Wiki](https://en.wikipedia.org/wiki/IEEE_754)，这里直接介绍单精度浮点数的表示：

<p align="center">
    <img src="/assets/understanding-the-fast-invsqrt-algorithm/ieee754_float_example.svg"/>
</p>

单精度浮点数由32位表示，分为三个部分：符号位$$S$$（1位）、指数位$$E$$（8位）和小数位$$M$$（23位）。

浮点数$$V$$表示公式如下：

$$
V = (-1)^S \times (1 + m) \times 2^e  \tag{2.4}
$$

其中：$$M = m \times L$$，$$E = e + B$$，$$L=2^{23}$$，$$B = 127$$。

如上图的例子所示：$$S=0$$，$$E=124$$，$$M=2^{21}=2097152$$，

所以：$$m = \frac{M}{L} = 0.25$$，$$e = E - B = -3$$，

所以：$$V = (1 + 0.25) \times 2^{-3} = 0.15625$$


## 3. 卡马克算法

代码：

```C
float Q_rsqrt( float number )
{
    long i;
    float x2, y;
    const float threehalfs = 1.5F;

    x2 = number * 0.5F;
    y  = number;
    i  = * ( long * ) &y;                       // evil floating point bit level hacking
    i  = 0x5f3759df - ( i >> 1 );               // what the fuck? 
    y  = * ( float * ) &i;
    y  = y * ( threehalfs - ( x2 * y * y ) );   // 1st iteration

    return y;
}
```

以上代码中有两行比较难理解：

```C
i  = 0x5f3759df - ( i >> 1 );               // what the fuck? 
```

以及

```C
y  = y * ( threehalfs - ( x2 * y * y ) );   // 1st iteration
```

### 3.1 牛顿迭代

我们首先理解这一行代码：

```C
y  = y * ( threehalfs - ( x2 * y * y ) );   // 1st iteration
```

对于浮点数$$a$$，我们构造一个函数$$f(x) = x^{-2} - a$$，当$$f(x) = 0$$时，得到的$$x$$即为答案（对应式$$2.2$$），首先对$$f(x)$$在$$x_k$$处进行一阶泰勒展开并令其为0：

$$
f(x) \approx f(x_k) + f'(x_k)(x-x_k) = 0  \tag{3.1}
$$

可以得到：

$$
x = x_k - \frac{f(x_k)}{f'(x_k)}  \tag{3.2}
$$

即利用函数在$$x_k$$的值和一阶导数的值即可进行一次牛顿法迭代。

将$$f(x_k) = x_k^{-2} - a$$，$$f'(x_k) = -2 x_k^{-3}$$代入式$$3.2$$，得到：

$$
\begin{aligned}
x &= \frac{3}{2} x_k - \frac{1}{2}x_k^3a  \\
&= x_k (\frac{3}{2} - \frac{1}{2}x_k^2 a)
\end{aligned}  \tag{3.3}
$$

代码中的$$y$$就是$$x_k$$，$$x2$$就是$$\frac{1}{2}a$$，这一行代码就是进行了一次牛顿法迭代。

这里就出现了一个疑问！为什么他只需要迭代一次？按道理不是应该迭代直到精度足够小吗？原因是**代码作者预先选择的出发点已经足够接近答案了**，另外一行神仙代码就是做这件事情。

### 3.2 初始值估计

下面理解这行神仙代码：

```C
i  = 0x5f3759df - ( i >> 1 );               // what the fuck? 
```

为什么这一行代码计算得到的值，就已经非常接近答案？以下分析参考自[Wiki](https://en.wikipedia.org/wiki/Fast_inverse_square_root)。

对于一个正浮点数$$x$$：

$$
x = 2^{e_x} (1 + m_x)  \tag{3.4}
$$

考虑其对数：

$$
\log x = e_x + \log (1 + m_x)  \tag{3.5}
$$

由于$$m_x \in [0, 1)$$，所以$$\log x$$可以表示为：

$$
\log x \approx e_x + m_x + \sigma  \tag{3.6}
$$

其中$$\sigma \approx 0.0430357$$。

接下来我们尝试表示出一个浮点数的整数部分，假设$$I_x$$表示浮点数$$x$$的整数部分（为什么整数部分是这么表示？笔者也未理解）：

$$
\begin{aligned}
I_x &= E_x L + M_x  \\
&= L (e_x + B + m_x)  \\
&= L (e_x + m_x + \sigma + B - \sigma)  \\
&\approx L \log x + L(B - \sigma) \\
\end{aligned}  \tag{3.7}
$$

进一步推导得到：

$$
\log x \approx \frac{I_x}{L} - (B - \sigma)  \tag{3.8}
$$

令$$y = \frac{1}{\sqrt{x}}$$，则$$\log y = -\frac{1}{2} \log x$$，代入式$$3.8$$：

$$
\log y \approx \frac{I_y}{L} - (B - \sigma) \approx -\frac{1}{2} (\frac{I_x}{L} - (B - \sigma))  \tag{3.9}
$$

化简得到：

$$
I_y = \frac{3}{2} (B-\sigma)L -\frac{1}{2}I_x  \tag{3.10}
$$

其中$$I_x$$表示浮点数$$x$$的整型部分，在代码中用强制转换得到；$$I_y$$表示$$\frac{1}{\sqrt{x}}$$的整数部分，作为牛顿迭代的初始值。

神仙代码就是实现了式$$3.10$$，其中$$(i>>1)$$表示$$\frac{1}{2}I_x$$（右移操作实现除2）；0x5f3759df作为$$\frac{3}{2}(B-\sigma)$$的估计。在当时计算条件这么有限的情况下，这个数字是怎么获得的呢？这要问作者本人了。

### 3.3 精度

这个算法得到的精度如何呢？图片来源于[Wiki](https://en.wikipedia.org/wiki/Fast_inverse_square_root)。

<p align="center">
    <img src="/assets/understanding-the-fast-invsqrt-algorithm/Invsqrt_acc.svg"/>
</p>

绿线表示算法得到的值，红线表示误差，可以看到误差随着数值的增大而减小，且恒定小于0.012。



## 参考

[Fast inverse square root](https://en.wikipedia.org/wiki/Fast_inverse_square_root)  \\
[IEEE 754](https://en.wikipedia.org/wiki/IEEE_754)  \\
[快速浮点开方运算](https://blog.csdn.net/yutianzuijin/article/details/78839981)
