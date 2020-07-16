---
layout: post
title: "Ensemble Learning I (Bagging and Boosting)"
date: 2020-07-15
categories: blog
permalink: /:categories/:year/:month/:title.html
---


## 1. Ensemble learning

集成学习本身是一种监督学习算法，它需要训练多个模型，然后将多个模型集成后的输出作为最终结果，集成后的模型的假设不一定被包含在构建它的模型的假设空间内。集成学习有很多好处，例如：

- 学习任务的假设空间可能很大，可能有多个假设在训练集上达到相同的性能，使用单学习器会导致泛化性能不佳；
- 从计算的角度看，学习算法可能在某次计算下陷入局部最小值，学习器结合可以有效避免这种情况带来的影响；
- 从表示的角度看，某些学习任务的真实假设不在当前学习算法考虑的假设空间中，因此单学习器会无效，学习器结合能得到更好的近似。

常见的集成方式有两种：Bagging和Boosting。其中Boosting方法中较为著名的有：AdaBoost和Gradient Boosting。

- Bagging
  - Random Forest
- Boosting
  - AdaBoost
  - Gradient Boosting
	- GBDT
	- XGBoost
	
（Formula部分提供理解本文推导所需的数学公式）

## 2. Bias-Variance Trade-off

$$
\begin{aligned}
Bias^2 &= (E[\hat h] - h)^2  \\
Var &= E[(\hat h - E[\hat h])^2]  \\
\epsilon^2 &= E[(y_D - y)^2]  \\
\end{aligned}  \tag{1}
$$

其中$$h$$表示真实模型，$$\hat h$$表示训练出的模型，$$y_D$$表示数据集标签，$$y$$表示真实标签，$$\epsilon$$是数据集的噪声。

- 偏差（$$Bias$$）
  - 偏差指训练出所有模型的输出的平均值与真实模型输出之间的偏差，刻画学计算法的拟合能力。
  - 通常是由于学习算法做了错误的假设导致的。
  - 描述模型输出结果的期望与样本真实结果的差距。
  
- 方差（$$Var$$）
  - 方差指训练出所有模型的输出的方差，刻画数据扰动造成的影响。
  - 描述模型的稳定性。
  
- 噪声（$$\epsilon$$）
  - 噪声表达当前任务上任何学习算法所能达到的期望泛化误差下界，刻画学习问题本身的难度。
  
> 泛化误差等于偏差和方差之和

Proof：
训练模型$$\hat h$$去逼近真实模型$$h$$

$$
\begin{aligned}
Err &= E[(h - \hat h)^2]  \\
&= E[h^2 - 2h \hat h + \hat h^2]  \\
&= E[h^2] - 2h E[\hat h] + E[\hat h ^2]  \\
&= h^2 - 2h E[\hat h] + E^2[\hat h] - E^2[\hat h] + E[\hat h ^2] \\
&= (E[\hat h] - h)^2 + (E[\hat h ^2] - E^2[\hat h])  \\
&= Bias^2 + Var
\end{aligned}  \tag{2}
$$

所以面对一个任务时，我们训练的模型必须既考虑偏差（准确度）也要考虑方差（泛化性）。

- 低偏差 + 低方差 = 良好模型
- 低偏差 + 高方差 = 过拟合
- 高偏差 + 低方差 = 欠拟合
- 高偏差 + 高方差 = 差模型

## 3. Bagging

### 3.1 基本概念

Bagging是指Bootstrap aggregating，学习器之间无依赖性，可以并行训练。

训练步骤：
1. 假设训练集$$D$$大小为$$N$$，从训练集中有放回地随机取出$$n$$个样本形成小训练集$$D'$$，共取出$$k$$个相互独立的小训练集；
2. 利用$$k$$个小训练集训练$$k$$个基学习器；
3. 训练完毕后，每个学习器的权重是一致的，最终采用投票的方式得到结果。

样本在$$n$$次采样过程中始终不会被采样得到的概率是$$(1-\frac{1}{n})^n$$，当$$n \rightarrow \infty$$时：

$$
\lim_{n \rightarrow \infty} (1 - \frac{1}{n})^n = \frac{1}{e} \approx 0.368  \tag{3}
$$

即每个基学习器使用了原始训练集大约$$63.8\%$$的样本。

> Bagging的优点：如果训练集中有噪声，那么每次采样都有机会将噪声样本剔除在外，有效降低模型的不稳定性。

### 3.2 对泛化误差的影响

Bagging方法的最终结果是由基学习器投票获得的，最终结果$$H$$的偏差表示为：

$$
\begin{aligned}
Bias[H] &= E[H] - h  \\
&= \frac{1}{k} \sum_{i=1}^k \hat h_i - h  \\
&= E[\hat h] - h
\end{aligned}  \tag{4}
$$

方差表示为（由于采样是有放回的，所以基学习器之间有一定的相关性，采用非独立变量的方差公式）：

$$
\begin{aligned}
Var[H] &= Var[\frac{1}{k} \sum_{i=1}^k \hat h_k]  \\
&= \frac{1}{k^2} Var[\sum_{i=1}^k \hat h_k]  \\
&= \frac{1}{k^2} [k \sigma^2 + (k-1)k \rho \sigma^2] \\
&= \frac{\sigma^2}{k} + \frac{k-1}{k} \rho \sigma^2
\end{aligned}  \tag{5}
$$

其中，假设基学习器方差均为$$\sigma^2$$，通过式$$(4)$$和$$(5)$$可以看出：
- Bagging的偏差是由基学习器的偏差决定的，所以Bagging方法的基学习器必须是是**强学习器**；
- 当增加基学习器数量时，Bagging能够**减少方差**，以达到减小泛化误差的目的。

## 4. RandomForest

随机森林是Bagging方法的代表，它的基学习器是CART树，训练流程采用Bagging的方法（采样、投票）。但是在构建树，分裂节点时稍有不同：在分裂节点上，随机选择$$K$$个特征，在选出来的特征中选择最优的特征进行分裂。每个$$h_i$$每次分裂时都对特征重新采样，有点类似于对特征再进行一次Bagging。这样能够使得随机森林中的决策树彼此尽可能不同，提升系统的多样性，从而提升分类性能。

## 5. Boosting

Boosting算法将多个弱基学习器合并成一个强学习器，它的做法是：

1. 首先对训练集的训练数据赋权重，初始情况下，权重当然是一样的；
2. 当一个学习器训练完毕后，训练集中仍然存在分类错误的数据，提升这些错误数据的权重；
3. 利用赋予了新权重的训练集训练下一个学习器，重复步骤2~3；
4. 当学习器的数量达到一定程度时，将所有的分类器收集起来，采用平均投票方式获得最终分类结果。

为什么最后需要对**所有**的分类器进行平均权重投票呢？

我个人的理解是，一开始训练的几个分类器的重心都在“容易分”的数据上，越往后训练，“困难数据”的权重越来越大，导致后面的分类器将中心都放在这些“困难数据”上，“容易分”的数据分类效果“可能”会下降，所以需要收集所有的分类器。由此可见，这种算法对噪声非常敏感，如果训练集中噪声比较多，那么后面的分类器都会集中在错误的噪声上，导致最终分类性能下降。

以上是最基本、最简单的Boosting思想，以此为基础还衍生出AdaBoost和Gradient Boosting。

### 5.1 AdaBoost

AdaBoost是Adaptive Boosting的缩写，它能够自适应各若基学习器的分类误差，具体做法为：

训练好第一个分类器后，错误率为：

$$
\epsilon_t = \sum_{i:h_t(x_i) \neq y_i} D_i^t  \tag{6}
$$

其中$$h_t$$为第$$t$$次训练好的分类器，$$i$$表示数据编号。

错误率一般来说小于0.5，定义该第$$t$$个模型的confidence为（可以从最小化训练误差边界来推导，也可以从最小化损失函数进行推导）：

$$
\alpha_t = \frac{1}{2} \log (\frac{1-\epsilon_t}{\epsilon_t})  \tag{7}
$$

接着，新的数据权重为：

$$
D_i^{t+1} = \exp (-y_i h_t(x_i)\alpha_t) D_i^t  \tag{8}
$$

最后对权重归一化：

$$
D_i^{t+1} = \frac{D_i^{t+1}}{\sum_{i=1}^N D_i^{t+1}}  \tag{9}
$$

最后如果要进行分类，新数据为$$x'$$：

$$
H(x') = sign[\sum_{t=1}^T \alpha_t h_t(x')]  \tag{10}
$$

### 5.2 迭代方式

Adaboost是一个加法模型，将过去训练好的基学习器按照权重相加：

$$
H(x) = \sum_{t=1}^T \alpha_t h_t(x; w_t)  \tag{11}
$$

最终目的其实是找到一组对应的权重$$\alpha$$和基学习器$$h$$，使得损失函数最小：

$$
\min_{\alpha, w} \sum_{i=1}^N L (y_i, \sum_{t=1}^T \alpha_t h_t(x; w_t))  \tag{12}
$$

但是这个过程是NP-hard的，所以用贪心的算法求解，每步只优化一个基学习器，再逐步迭代。

### 5.3 对泛化误差的影响

Boosting方法的最终结果是由基学习器按权重投票获得的，最终结果$$H$$的偏差表示为：

$$
\begin{aligned}
Bias[H] &= E[H] - h  \\
&= \sum_{i=1}^k \alpha E[\hat h_i] - h
\end{aligned}  \tag{13}
$$

方差表示为（基学习器之间是强相关性的，相关系数$$\rho$$近似为1，同样使用非独立变量方差的公式）：

$$
\begin{aligned}
Var[H] &= Var[\sum_{i=1}^k \alpha \hat h_k]  \\
&= k \alpha^2 \sigma^2 + (n-1)n \alpha^2 \sigma^2  \\
&= k^2 \alpha^2 \sigma^2
\end{aligned}  \tag{14}
$$

结合式$$(13)$$和$$(14)$$，
- 增加基学习器可使得偏差更小，但是会提升方差；
- 所以要使用**弱学习器**。


## Formula

方差与期望的关系：

$$
\begin{aligned}
\sigma^2 &= E[(X-\mu)^2] \\
&= E[X^2 - 2\mu X + \mu^2]  \\
&= E(X^2) - 2\mu E(X) + \mu^2  \\
&= E(X^2) - E^2[X]
\end{aligned}
$$

独立变量的方差：

$$
Var(ax + by) = a^2 Var(x) + b^2 Var(y)
$$

非独立变量的方差（$$\sigma^2 = Var(X_i)$$，$$\rho$$为变量之间的相关系数。如果$$X_i$$前面有系数$$\alpha$$，$$\alpha^2 \sigma^2 = Var(\alpha X_i)$$）：

$$
Var(\sum_{i=1}^n X_i) = n \sigma^2 + (n-1)n \rho \sigma^2
$$

协方差：

$$
\begin{aligned}
Cov(X, Y) &= E[(X - \mu_X)(Y - \mu_Y)]  \\
&= E(XY) - E(X)E(Y)
\end{aligned}
$$

线性组合的协方差：

$$
Cov(\sum_{i=1}^m a_i x_i, \sum_{j=1}^n b_j y_j) = \sum_{i=1}^m \sum_{j=1}^n a_i b_j Cov(x_i, y_j)
$$

两个随机变量之间的相关系数：

$$
Corr(X, Y) = \frac{Cov(X, Y)}{\sqrt{Var(X)Var(Y)}}
$$

## 参考

[Understanding the Bias-Variance Tradeoff](http://scott.fortmann-roe.com/docs/BiasVariance.html)  \\
[Bagging与方差](https://zhuanlan.zhihu.com/p/36822575)  \\
[集成学习（二）Boosting与GBDT](https://www.hrwhisper.me/machine-learning-model-ensemble-boostring-and-gbdt/)
