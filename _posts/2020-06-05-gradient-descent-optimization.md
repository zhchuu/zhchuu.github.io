---
layout: post
title: "Gradient Descent Optimization"
date: 2020-06-05
categories: blog
permalink: /:categories/:year/:month/:title.html
---


## 1. 训练方式
### 1.1 Batch gradient descent
计算**整个训练集**的梯度，进行一次梯度下降优化。

### 1.2 Stochastic gradient descent (SGD)
对于**每个样本**计算梯度，进行一次梯度下降优化。

### 1.3 Mini-batch gradient descent
将整个训练集分为多个**不重合的批次**，计算单个批次的梯度，进行一次梯度下降优化。

## 2. 优化器
### 2.1 Simple SGD

$$
\theta = \theta - \eta \bigtriangledown_{\theta} J(\theta) \tag{1}
$$

### 2.2 SGD with momentum
Momentum是一种加速SGD并抑制振荡的方法，它将上一次更新的方向也纳入本次更新的考虑项中：

$$
\begin{aligned}
v_t &= \gamma v_{t-1} + \eta\bigtriangledown_\theta J(\theta)  \\
\theta &= \theta - v_t
\end{aligned} \tag{2}
$$

如果更新与动量方向相同，那么会加速；如果与动量方向相反，会减速；

### 2.3 Nesterov accelerated gradient (NAG)
即使加上了momentum，梯度更新方向仍然会振荡。首先我们知道，如果仍然使用momentum，下一次更新**大约**方向为$$\gamma v_{t-1}$$，更新后的参数为$$\theta - \gamma v_{t-1}$$。NAG做的事情是**假设已知更新后的参数，计算更新后的参数的梯度**。

$$
\begin{aligned}
v_t &= \gamma v_{t-1} + \eta \bigtriangledown_\theta J(\theta - \gamma v_{t-1}) \\ 
\theta &= \theta - v_t
\end{aligned} \tag{3}
$$

### 2.4 Adagrad
> It adapts the learning rate to the parameters, performing smaller updates
(i.e. low learning rates) for parameters associated with frequently occurring features, and larger updates (i.e. high learning rates) for parameters associated with infrequent features. For this reason, it is well-suited for dealing with sparse data.

本质思想是对于不同的参数使用不同的学习率。假设$$g_{t, i}$$表示在$$t$$时刻$$\theta_i$$的偏导。

$$
g_{t, i} = \bigtriangledown_\theta J(\theta_{t, i}) \tag{4}
$$

对于SGD：

$$
\theta_{t+1, i} = \theta_{t, i} - \eta g_{t, i} \tag{5}
$$

对于Adagrade：

$$
\theta_{t+1, i} = \theta_{t, i} - \frac{\eta}{\sqrt{G_{t, ii} + \epsilon}} g_{t, i} \tag{6}
$$

其中$$G_t \in \mathbb{R}^{d \times d}$$是一个对角矩阵。对角线$$(i ,i)$$上的值是$$\theta_i$$在$$1 \sim t$$时刻梯度的平方和。$$\epsilon$$是平滑因子，防止分母为0。上式可以写成：

$$
\theta_{t+1} = \theta_{t} - \frac{\eta}{\sqrt{G_{t} + \epsilon}} \odot g_{t} \tag{7}
$$

优点：自适应学习率，分母相当于对学习率进行了自动调整。能够更好地应用于稀疏梯度的场景。

缺点：由于$$G_t$$矩阵中的值是累加的，所以随着时间退役，更新趋向于无穷小。

### 2.5 Adadelta
Adadelta是Adagrad的优化版，它用窗口限制住过往梯度的求和范围到一个固定的大小$$w$$。

但是直接设置$$w$$又不太好，所以算法的实现是对过去所有的梯度更新的平均值逐渐降低。
> Instead of inefficiently storing w previous squared gradients, the sum of gradients is recursively defined as a decaying average of all past squared gradients.

$$
E[g^2]_t = \gamma E[g^2]_{t-1} + (1-\gamma)g_t^2  \tag{8}
$$

当前时刻$$t$$估计均值$$E[g^2]_t$$是由时刻$$t-1$$的均值与当前的梯度决定的。

将式$$(8)$$替换式$$(7)$$的$$G_t$$：

$$
\theta_{t+1} = \theta_{t} - \frac{\eta}{\sqrt{E[g^2]_t + \epsilon}} \odot g_{t} \tag{9}
$$

观察上式，分母是均方根，所以缩写为：

$$
\theta_{t+1} = \theta_{t} - \frac{\eta}{RMS[g]_t} \odot g_{t} \tag{10}
$$

这样更新仍然是依赖于全局学习率的，作者进一步处理，经过近似牛顿迭代法后，将分子替换成参数的平方更新：

$$
RMS[\Delta \theta]_t = \sqrt{E[\Delta \theta^2]_t + \epsilon}  \tag{11}
$$

最终的更新为：

$$
\theta_{t+1} = \theta_{t} - \frac{RMS[\Delta \theta]_{t-1}}{RMS[g]_t} \odot g_{t} \tag{12}
$$

### 2.6 RMSprop
RMSprop是未发表的、*Geoff Hinton*在课堂上提到的一种自适应学习率的方法。与Adadelta中的式$$(9)$$是一样的。

### 2.7 Adam (Adaptive Moment Estimation)
同样是一种自适应学习率的更新方法。它估计梯度的一阶（均值）和二阶（未中心化的方差）。

Adam记录过去梯度的指数下降平均$$m_t$$，以及梯度平方的指数下降平均$$v_t$$：
> We compute the decaying averages of past and past squared gradients $$m_t$$ and $$v_t$$ respectively as follows:

$$
\begin{aligned}
m_t &= \beta_1 m_{t-1} + (1-\beta_1) g_t  \\
v_t &= \beta_2 v_{t-1} + (1-\beta_2) g_t^2
\end{aligned} \tag{13}
$$

由于$$m_0$$和$$v_0$$初始化为0，导致训练阶段偏向于0，进行纠正：

$$
\begin{aligned}
\hat{m}_t &= \frac{m_t}{1-\beta_1^t}  \\
\hat{v}_t &= \frac{v_t}{1-\beta_2^t}
\end{aligned} \tag{14}
$$

最终更新：

$$
\theta_{t+1} = \theta_t - \frac{\eta}{\sqrt{\hat{v}_t} + \epsilon} \hat{m}_t  \tag{15}
$$

个人看法：实际上相比于AdaDelta和RMSprop多了一个估计$$g_t$$的过程，用$$m_t$$来近似。

主要包含以下几个显著的优点：
1. 实现简单，计算高效，对内存需求少
2. 参数的更新不受梯度的伸缩变换影响
3. 超参数具有很好的解释性，且通常无需调整或仅需很少的微调
4. 更新的步长能够被限制在大致的范围内（初始学习率）
5. 能自然地实现步长退火过程（自动调整学习率）
6. 很适合应用于大规模的数据及参数的场景
7. 适用于不稳定目标函数
8. 适用于梯度稀疏或梯度存在很大噪声的问题

## Reference
[An overview of gradient descent optimization algorithms](https://ruder.io/optimizing-gradient-descent/)

[Adam](https://www.jianshu.com/p/aebcaf8af76e)
