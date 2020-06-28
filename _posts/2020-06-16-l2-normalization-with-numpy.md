---
layout: post
title: "L2 normalization with NumPy"
date: 2020-06-16
categories: blog
permalink: /:categories/:year/:month/:title.html
---

Size为$$(n, m)$$的NumPy数组，现在要对**行**做L2归一化。对于**每一行**：

$$
X_i = [x_1, x_2, ..., x_n] \\
$$

进行$L2$归一化：

$$
||X_{i}|| = \sqrt{\sum_{k=1}^{n}x_k^{2}} \\
L_2(X_i) = [\frac{x_1}{||X_{i}||}, \frac{x_2}{||X_{i}||}, ..., \frac{x_n}{||X_{i}||}]
$$

下面使用PyTorch和NumPy都操作一次：
```Python
>>> import numpy as np
>>> import torch
>>> import torch.nn.functional as F
```


## 1. 单行操作：
### 1.1 PyTorch
```Python
>>> arr = torch.Tensor([[1, 2, 3, 4]])
>>> F.normalize(arr, p=2)  # Use PyTorch
tensor([[0.1826, 0.3651, 0.5477, 0.7303]])
```

### 1.2 NumPy
```Python
>>> arr = arr.numpy()
>>> arr / np.linalg.norm(arr)  # Use NumPy
array([[0.18257418, 0.36514837, 0.5477225 , 0.73029673]], dtype=float32)
```

## 2. 对多行进行操作：
### 2.1 PyTorch
```Python
>>> arr = torch.randn(3, 4)
>>> arr
tensor([[ 0.7112,  0.6966, -1.0856,  0.4245],
        [ 1.0093,  0.1368,  1.0226,  1.4529],
        [ 0.5584, -0.8238, -1.4474, -0.3137]])
>>> F.normalize(arr, p=2, dim=1)  # Use PyTorch
tensor([[ 0.4640,  0.4544, -0.7082,  0.2769],
        [ 0.4928,  0.0668,  0.4993,  0.7095],
        [ 0.3130, -0.4617, -0.8112, -0.1758]])
```

### 2.2 NumPy
```Python
>>> arr = arr.numpy()
>>> arr / np.linalg.norm(arr, axis=1) # Wrong operation
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
ValueError: operands could not be broadcast together with shapes (3,4) (3,)
```
```Python
>>> (arr.T / np.linalg.norm(arr.T, axis=0)).T  # Correct operation
array([[ 0.46395493,  0.4544233 , -0.7082126 ,  0.27691197],
       [ 0.49284512,  0.06680366,  0.4993104 ,  0.7094576 ],
       [ 0.3129517 , -0.46167102, -0.8111806 , -0.17580412]],
      dtype=float32)
```

当对多行进行操作时，
```Python
np.linalg.norm(arr, axis=1)
```
能正确计算**针对每行进行L2 Regularization**，此时得到的NumPy数组的Size为$$(3,)$$，原数组的Size为$$(3, 4)$$。所以要转置，相除，再转置。

以下两种操作得到的结果应该是一致的：

```Python
>>> np.linalg.norm(arr.T, axis=0)
array([1.5329131, 2.0479364, 1.7842926], dtype=float32)
>>> np.linalg.norm(arr, axis=1)
array([1.5329131, 2.0479364, 1.7842926], dtype=float32)
```
