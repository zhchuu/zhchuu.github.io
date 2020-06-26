---
layout: post
title: "Product Operations of Matrix and Tensor"
date: 2020-06-26
categories: blog
permalink: /:categories/:year/:month/:title.html
---

Summary of the product operations(e.g., dot product, matrix product, product with batch) with NumPy and PyTorch.

## 1. Matrix

### 1.1 Element-wise product

$$(m, n) \odot (m, n) = (m, n)$$

**Initial data:**
```Python
a = torch.Tensor([[1, 1, 1], 
                  [2, 2, 2]])
b = torch.Tensor([[3, 3, 3], 
                  [0, 0, 0]])
c = torch.Tensor([[2, 2, 2], 
                  [2, 2, 2], 
                  [2, 2, 2]])
print(a.shape)
print(b.shape)
print(c.shape)
```
```Python
torch.Size([2, 3])
torch.Size([2, 3])
torch.Size([3, 3])
```

**Method 1:**
```Python
a * b
```
```Python
tensor([[3., 3., 3.],
        [0., 0., 0.]])
```

**Method 2:**
```Python
torch.mul(a, b)
```
```Python
tensor([[3., 3., 3.],
        [0., 0., 0.]])
```

**Method 3:**
```Python
a.numpy() * b.numpy()
```
```Python
array([[3., 3., 3.],
       [0., 0., 0.]], dtype=float32)
```

**Method 4:**
```Python
np.multiply(a.numpy(), b.numpy())
```
```Python
array([[3., 3., 3.],
       [0., 0., 0.]], dtype=float32)
```


### 1.2 Matrix product

$$(m, n) \circ (n, k) = (m, k)$$

**Initial data:**
```Python
print(a.shape)
print(b.shape)
print(c.shape)
```
```Python
torch.Size([2, 3])
torch.Size([2, 3])
torch.Size([3, 3])
```

**Method 1:**
```Python
x = a.numpy().dot(c.numpy())
print(x)
print(x.shape)
```
```Python
[[ 6.  6.  6.]
 [12. 12. 12.]]
(2, 3)
```

**Method 2:**
```Python
x = np.matmul(a.numpy(), c.numpy())
print(x)
print(x.shape)
```
```Python
[[ 6.  6.  6.]
 [12. 12. 12.]]
(2, 3)
```

**Method 3:**
```Python
x = torch.mm(a, c)
print(x)
print(x.shape)
```
```Python
tensor([[ 6.,  6.,  6.],
        [12., 12., 12.]])
torch.Size([2, 3])
```

**Method 4:**
```Python
x = torch.matmul(a, c)
print(x)
print(x.shape)
```
```Python
tensor([[ 6.,  6.,  6.],
        [12., 12., 12.]])
torch.Size([2, 3])
```

**with Python 3.5+**

**Method 5:**
```Python
a @ c
```
```Python
tensor([[ 6.,  6.,  6.],
        [12., 12., 12.]])
```

```Python
a.numpy() @ c.numpy()
```
```Python
array([[ 6.,  6.,  6.],
       [12., 12., 12.]], dtype=float32)
```


## 2. Tensor

### 2.1 Element-wise product

$$(m, n, k) \odot (m, n, k) = (m, n, k)$$

**Initial data:**
```Python
a = torch.randn(32, 3, 3)
b = torch.randn(3, 5)
c = torch.randn(32, 3, 6)
print(a.shape)
print(b.shape)
print(c.shape)
```
```Python
torch.Size([32, 3, 3])
torch.Size([3, 5])
torch.Size([32, 3, 6])
```

**Method 1:**
```Python
(a * a).shape
```
```Python
torch.Size([32, 3, 3])
```

**Method 2:**
```Python
torch.mul(a, a).shape
```
```Python
torch.Size([32, 3, 3])
```

**Method 3:**
```Python
(a.numpy() * a.numpy()).shape
```
```Python
(32, 3, 3)
```

**Method 4:**
```Python
np.multiply(a.numpy(), a.numpy()).shape
```
```Python
(32, 3, 3)
```

### 2.2 Tensor product with batch I 

$$(b, m, n) \circ (n, k) = (b, m, k)$$

**Method 1:**
```Python
a.numpy().dot(b.numpy()).shape
```
```Python
(32, 3, 5)
```

**Method 2:**
```Python
torch.matmul(a, b).shape
```
```Python
torch.Size([32, 3, 5])
```

**with Python 3.5+**

**Method 3:**
```Python
(a @ b).shape
```
```Python
torch.Size([32, 3, 5])
```

### 2.3 Tensor product with batch II

$$(b, m, n) \circ (b, n, k) = (b, m, k)$$

**Method 1:**
```Python
np.matmul(a.numpy(), c.numpy()).shape
```
```Python
(32, 3, 6)
```

**Method 2:**
```Python
torch.matmul(a, c).shape
```
```Python
torch.Size([32, 3, 6])
```

**Method 3:**
```Python
torch.bmm(a, c).shape
```
```Python
torch.Size([32, 3, 6])
```

**with Python 3.5+**

**Method 4:**
```Python
(a @ c).shape
```
```Python
torch.Size([32, 3, 6])
```
