---
layout: post
title: "Relationship between L1 / L2 Regularization and Parameters Prior"
date: 2020-06-27
categories: blog
permalink: /:categories/:year/:month/:title.html
---

> From a Bayesian persipective, if parameters are given a Gaussian prior, this yields L2 regularisation (or weight decay). If parameters are given a Laplace prior, then L1 regularisation is recovered.

## 1. L2 regularization
> If parameters are given a Gaussian prior, this yields L2 regularisation (or weight decay).

Proof:

Assume that we want to infer some paramter $$\beta$$ from some observed input-output pairs $$(x_1, y_1), (x_2, y_2), \cdots, (x_N, y_N)$$。Let us assume that the outputs are linearly related to the inputs via $$\beta$$ and that the data are corrupted by some noise $$\epsilon$$:

$$
y_n = \beta x_n + \epsilon \tag{1}
$$

where $$\epsilon$$ is Gaussian noise with mean 0 and variance $$\sigma^2$$. This gives rise to a Gaussian likelihood:

$$
\prod_{n=1}^N \cal{N}(y_n|\beta x_n, \sigma^2)  \tag{2}
$$

Let us regularize parameter $$\beta$$ by imposing the Gaussian prior $$\cal{N}(\beta \vert 0, \lambda^{-1})$$, where $$\lambda$$ is a strictly positive scalar. Hence, combining the likelihood and the prior:

$$
\prod_{n=1}^N \cal{N}(y_n|\beta x_n, \sigma^2) \cal{N}(\beta|0, \lambda^{-1})  \tag{3}
$$

Take the logarithm of the above expression:

$$
\log(\frac{1}{\sqrt{2\pi}\sigma}) + \log(\frac{1}{\sqrt{2\pi \lambda^{-1}}}) + \sum_{n=1}^N \frac{-(y_n - \beta x_n)^2}{2 \sigma^2} + \frac{-\beta^2 \lambda^2}{2}  \tag{4}
$$

Dropping some constants:

$$
\sum_{n=1}^N -\frac{1}{\sigma^2} (y_n - \beta x_n)^2 - \beta^2 \lambda^2 + const  \tag{5}
$$

Thus, this is equivalent to inducing priors on the weights (say Gaussian distributions if we are using L2 regularization).

## 2. L1 regularization
> If parameters are given a Laplace prior, then L1 regularisation is recovered.

Proof:

If we regularize parameter $$\beta$$ by Laplace prior $$Laplace(0, b)$$. Hence, combining the likelihood and the prior:

$$
\prod_{n=1}^N \cal{N}(y_n|\beta x_n, \sigma^2) Laplace(\beta|0, b)  \tag{6}
$$

Take the logarithm of the above expression:

$$
\log(\frac{1}{\sqrt{2\pi}\sigma}) + log(\frac{1}{2b}) + \sum_{n=1}^N \frac{-(y_n - \beta x_n)^2}{2 \sigma^2} + \frac{- \vert \beta \vert}{b}  \tag{7}
$$

Dropping some constants:

$$
\sum_{n=1}^N -\frac{1}{\sigma^2} (y_n - \beta x_n)^2 - \frac{ \vert \beta \vert}{b} + const  \tag{8}
$$
