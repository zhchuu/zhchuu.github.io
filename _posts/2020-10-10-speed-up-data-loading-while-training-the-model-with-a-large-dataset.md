---
layout: post
title: "Speed up data loading while training the model with a large dataset"
date: 2020-10-10
categories: blog
permalink: /:categories/:year/:month/:title.html
---


## 1. 前言

在使用大规模数据（例如ImageNet1000）训练深度神经网络时，**I/O**往往会成为训练的瓶颈。以ILSVRC2012为例，数据集包含1000类，训练集大约有一百二十八万张图片，如果全部以$$224 \times 224 \times \times 3$$的图像读入内存，每个像素以256位比特存储，那么理论上将占用$$1280000 \times 224 \times 224 \times 3 \times 1 Bytes \approx 192 GB$$的内存。这显然不是所有机器都能胜任的数量级。所以对于这样的数据集一般的做法是：**存储样本的路径，每次加载的时候读取出来。**

既然如此，那么必然会涉及到大量的读取操作，就很有可能出现一种情况：显卡没有满载，但CPU却忙于读取数据。为了避免这种情况的发生，以下几种方法是可以参考的。

## 2. 提升数据加载速度

### 2.1 accimage

> accimage是Pytorch团队写的，基于JPEG-Turbo以及Intel IPP的一个图像库，仅支持RGB的JPEG的图像，可以用于部分替代PIL.Image。

#### 安装：
> $ conda install -c conda-forge accimage

#### 使用：

启用后端：
> torchvision.set_image_backend('accimage')

可通过以下方法检查是否启用正确：
```C
>>> import torchvision
>>> from torchvision import get_image_backend
>>> get_image_backend()
'PIL'
>>> torchvision.set_image_backend('accimage')
>>> get_image_backend()
'accimage'
>>> 
```

以下代码参考自：[Pytorch源码](https://github.com/pytorch/vision/blob/master/torchvision/datasets/folder.py#L174)

```Python
def accimage_loader(path):
    import accimage
    from PIL import Image
    try:
        return accimage.Image(path)
    except IOError:
        # Potentially a decoding problem, fall back to PIL.Image
        img = Image.open(path)
        return img.convert('RGB')
```

#### 优缺点：

优点：
1. 在某些场景下可以无缝替换PIL.Image；
2. 速度真的比PIL.Image快很多；
3. torchvision中的ImageFolder方法里，获得的Dataset会自动使用accimage库读取图像，只需要按照上述步骤正确安装accimage，且打开后端。

缺点：
1. 仅支持JPEG图像，目前没有对PNG的支持；
2. 仅支持conda安装，不支持pip安装。


### 2.2 Pillow-SIMD

> Pillow-SIMD是基于Pillow的，并且可以100%兼容、替换原来版本Pillow的图像库，可以理解为Pillow的加速版。SIMD代表：single instruction, multiple data。对大量数据进行同一种操作时，Pillow-SIMD会更快。

#### 安装：
> $ pip uninstall pillow \\
> $ CC="cc -mavx2" pip install -U --force-reinstall pillow-simd

#### 使用：
不用改任何代码。

#### 优缺点：

优点：
1. 完美替换PIL.Image；
2. 速度真的比PIL.Image快很多；

缺点：
1. 初步使用下来，基本没有发现什么缺点，安装完成后代码都不需要修改。但是要注意安装的版本是否兼容。


### 3. 提升数据传输速度

除了在读取数据时候进行加速，Pytorch还提供一个小的数据传输优化方式。

### 3.1 pin_memory

在Pytorch的DataLoader中，提供了一个参数叫：pin_memory，默认为False。具体原理可以移步到[这里](https://developer.nvidia.com/blog/how-optimize-data-transfers-cuda-cc/)查看。

![](/assets/speed-up-data-loading-while-training-the-model-with-a-large-dataset/pin_memory.jpg)

我的理解为：

当CPU有数据要传输给GPU时：由于CPU是Pageable Memory的形式存储数据，需要先创建一个Pinned Memory区域，把数据拷贝到Pinned Memory区域，再将数据传输到DRAM中，最终从DRAM传输给GPU。其中在CPU拷贝的过程就多余了，所以设置pin_memory=True后，CPU直接创建Pinned Memory为数据锁定区域，这样就能直接传输给DRAM，减少数据传输时间。

当然这么做也是有不好的地方的：直接创建Pinned Memory意味着这段缓存不能被替换，如果这样的缓存太多了，会导致其他数据的Hit rate大幅下降，进而可能导致卡顿、死机的状况。所以pin_memory的设置会要求机器有较好的性能，这也是这个字段的defalut为False的原因。

在通过样本路径加载的方式下，一般会把pin_memory设置为True。

### 4. 总结

个人简单地对上述方法进行了尝试，从速度上讲，**个人感觉**：

$$ pin\_memory=True > pin\_memory=False $$

$$ accimage \approx PillowSIMD >> Pillow$$

accimage和Pillow-SIMD的速度差不多，且都优于Pillow。


## 参考

[What is accimage?](https://github.com/pytorch/accimage/issues/12)  \\
[Pillow-SIMD](https://github.com/uploadcare/pillow-simd)  \\
[How to Optimize Data Transfers in CUDA C/C++](https://developer.nvidia.com/blog/how-optimize-data-transfers-cuda-cc/)  \\
[torch.utils.data](https://pytorch.org/docs/stable/data.html)
