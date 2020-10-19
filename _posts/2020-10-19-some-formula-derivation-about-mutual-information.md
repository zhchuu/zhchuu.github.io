---
layout: post
title: "Some Formula Derivation about Mutual Information"
date: 2020-10-19
categories: blog
permalink: /:categories/:year/:month/:title.html
---


## 1. 熵（Entropy）

在进行公式推导前，有必要将信息熵的一些基本概念和公式复习一遍。

### 1.1 信息熵（Information entropy）

一个随机变量$$X$$的信息熵通常用符号$$H(X)$$表示。

$$
H(X) = - \sum_x p(x) \log p(x)  \tag{1}
$$

熵是对随机变量不确定性的度量，信息熵越大，混乱程度越大，不确定性越大。香农编码定理表明熵是传输一个随机变量状态值所需的比特位下界（最短平均编码长度）。

两个随机变量$$X$$和$$Y$$的联合熵（Joint entropy）表示为：

$$
H(X, Y) = - \sum_{x, y} p(x, y) \log p(x, y)  \tag{2}
$$

### 1.2 条件熵（Conditional entropy）

条件熵$$H(Y \vert X)$$表示，在已知随机变量$$X$$的前提下随机变量$$Y$$的不确定性：

$$
\begin{aligned}
H(Y \vert X) &= \sum_x p(x) H(Y \vert X=x)  \\
&= - \sum_x p(x) \sum_y p(y \vert x) \log p(y \vert x) \\
&= - \sum_x \sum_y p(x) p(y \vert x) \log p(y \vert x) \\
&= - \sum_{x,y} p(x, y) \log p(y \vert x)
\end{aligned}  \tag{3}
$$

条件熵$$X(Y \vert X)$$可以表示为联合熵$$H(X, Y)$$减去信息熵$$H(X)$$，即$$X(Y \vert X) = H(X, Y) - H(X)$$，证明：

$$
\begin{aligned}
H(X, Y) &= - \sum_{x,y} p(x, y) \log p(x, y) \\
&= - \sum_{x, y} p(x, y) \log (p(y \vert x) p(x))  \\
&= - \sum_{x, y} p(x, y) \log p(y \vert x) - \sum_{x, y} p(x, y) \log p(x)  \\
&= H(Y \vert X) - \sum_{x, y} p(x, y) \log p(x)  \\
&= H(Y \vert X) - \sum_x \sum_y p(x, y) \log p(x) \\
&= H(Y \vert X) - \sum_x \log p(x) \sum_y p(x, y) \\
&= H(Y \vert X) - \sum_x \log p(x) p(x) \\
&= H(Y \vert X) + H(X)
\end{aligned}  \tag{4}
$$

可以理解为：在$$X$$和$$Y$$非独立分布的条件下，当已知$$X$$会发生时，那么$$Y$$取值的不确定性减小了。

等价地：$$H(X \vert Y) = H(X, Y) - H(Y)$$

### 1.3 相对熵（Relative entropy）

相对熵也就是$$KL$$散度，具体公式在[变分推导](https://zhchuu.github.io/blog/2020/05/understanding-variational-inference.html)的第一部分中有详细解释。

### 1.4 交叉熵（Cross entropy）

交叉熵是通过$$KL$$散度推导而来的，具体公式在[Understanding CrossEntropy](https://zhchuu.github.io/blog/2020/05/understanding-cross-entropy.html)中有详细解释。


## 2. 互信息（Mutual information）

在概率论和信息论中，两个随机变量的互信息是它们之间**依赖性**的度量。

### 2.1 定义

两个离散随机变量$$X$$和$$Y$$的互信息定义为：

$$
I(X; Y) = \sum_y \sum_x p(x, y) \log \frac{p(x, y)}{p(x) p(y)}  \tag{5}
$$

其中$$p(x, y)$$表示联合概率分布，$$p(x)$$和$$p(y)$$表示边缘概率分布。

对于连续随机变量，互信息定义为：

$$
I(X; Y) = \int_y \int_x p(x, y) \log \frac{p(x, y)}{p(x) p(y)} dx dy  \tag{6}
$$

举个简单的例子：对于两枚一模一样的正常硬币$$A$$和$$B$$，抛出正面的概率都是$$\frac{1}{2}$$，他们各自抛一次，计算抛出都是正面的互信息。

$$
I(A^+; B^+) = p(A^+, B^+) \log \frac{p(A^+, B^+)}{p(A^+) \times p(B^+)} = \frac{1}{4} \log \frac{\frac{1}{4}}{\frac{1}{2} \times \frac{1}{2}} = 0
$$

说明，**抛这两枚硬币没有任何依赖性**，已知第一枚硬币抛到正面或反面，都不会影响第二枚抛出什么。

由上述的公式可以看出：**当$$X$$和$$Y$$相互独立时，即$$p(x, y) = p(x) p(y)$$，互信息为0。**

### 2.2 常见等价公式及推导

除了式$$(5)$$和式$$(6)$$可以表示互信息，还有其他表示形式。

利用$$KL$$散度来表示：

$$I(X; Y) = KL(p(x, y) || p(x) p(y))  \tag{7}$$

此外，另$$p(x \vert y) = \frac{p(x, y)}{p(y)}$$，则：

$$
\begin{aligned}
I(X; Y) &= \sum_y \sum_x p(x \vert y) p(y) \log \frac{p(x \vert y)}{p(x)} \\
&= \sum_y p(y) \sum_x p(x \vert y) \log \frac{p(x \vert y)}{p(x)} \\
&= \sum_y p(y) KL(p(x \vert y) || p(x)) \\
&= \mathbb{E}_Y \{KL(p(x \vert y) || p(x))\}
\end{aligned}  \tag{8}
$$

还可以利用熵来表示：

$$
\begin{aligned}
I(X; Y) &= \sum_{x, y} p(x, y) \log \frac{p(x, y)}{p(x) p(y)} \\
&= \sum_{x, y} p(x, y) \log \frac{p(x, y)}{p(x)} - \sum_{x, y} p(x, y) \log p(y)  \\
&= \sum_{x, y} p(x) p(y \vert x) \log p(y \vert x) - \sum_{x, y} p(x, y) \log p(y)  \\
&= \sum_x p(x) (\sum_y p(y \vert x) \log p(y \vert x)) - \sum_y \log p(y) (\sum_x p(x, y)) \\
&= - \sum_x p(x) H(Y \vert X=x) - \sum_y p(y) \log p(y) \\
&= - H(Y \vert X) + H(Y)  \\
&= H(Y) - H(Y \vert X)
\end{aligned}  \tag{9}
$$

由互信息是对称的，即$$I(X; Y) = I(Y; X)$$，所以：

$$
\begin{aligned}
I(X; Y) &= H(Y) - H(Y \vert X)  \\
&= H(X) - H(X \vert Y)
\end{aligned}  \tag{10}
$$

由式$$(4)$$可知：$$H(X \vert Y) = H(X, Y) - H(Y)$$，代入式$$(10)$$可得：

$$
\begin{aligned}
I(X; Y) &= H(X) + H(Y) - H(X, Y)  \\
&= H(X, Y) - H(Y \vert X) - H(X \vert Y)
\end{aligned}  \tag{11}
$$

### 2.3 性质

1. 对称性：
$$I(X; Y) = I(Y; X)$$

2. 非负性：
$$
I(X; Y) \geqslant 0
$$

3. 极值性：
$$
\begin{aligned}
&I(X; Y) \leqslant H(X)  \\
&I(Y; X) \leqslant H(Y)
\end{aligned}
$$
当$$X$$与$$Y$$一一对应时取等号。

## 参考

[Mutual information](https://en.wikipedia.org/wiki/Mutual_information)  \\
[互信息（Mutual Information）](https://blog.csdn.net/bbbeoy/article/details/72571902)  \\
[详解机器学习中的熵、条件熵、相对熵和交叉熵](https://zhuanlan.zhihu.com/p/35379531) \\
[互信息的公式推导](http://nathanlvzs.github.io/Mutual-Information-Formula-Derivation.html)
