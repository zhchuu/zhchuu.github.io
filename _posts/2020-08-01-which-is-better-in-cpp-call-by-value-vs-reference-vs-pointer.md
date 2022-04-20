---
layout: post
title: "Which is Better in C++? Call by Value, Reference or Pointer"
date: 2020-08-01
categories: blog
permalink: /:categories/:year/:month/:title.html
---


## 1. 问题由来

前段时间我用C++做算法题目时遇到一题出现有意思的现象，算法本身没有问题但是一直出现TLE错误。后来我将函数传入的参数改为值的引用，代码不仅被Accepted且运行速度击败了89.9%的人。这令我思考，C++调用函数时，传入参数为值、引用、指针对速度有什么影响？

程序TLE：
```C
void func(vector<int> val1, string val2){

}
```

程序Accepted：
```C
void func(vector<int>& val2, string& val2){

}
```

## 2. Pointer vs. Reference

**指针与引用的区别**：

- 指针（Pointer）：一个存储着变量的内存地址的值。读取时需要先读取地址再获得变量的值，所以要用*符号引导。
- 引用（Reference）：已经存在的变量的别名。从实现的角度看，也是记录变量的内存地址，所以相当于const pointer（编译器自动使用*引导）。

根据定义可知，不考虑编译器优化、内存管理差别等因素，**在调用函数时使用指针或引用速度上差别不大，但指针比引用多一次地址复制**。

如下代码所示：
```C
#include <iostream>

void callByPointer(int* x) {
  *x = 999;
  std::cout << &x << " " << x << " " << *x << std::endl;
}

void callByReference(int*& x) {
  *x = 888;
  std::cout << &x << " " << x << " " << *x << std::endl;
}

int main() {
  int *x = new int();

  std::cout << "Call by pointer:" << std::endl;
  *x = 0;
  std::cout << &x << " " << x << " " << *x << std::endl;
  callByPointer(x);

  std::cout << "\nCall by reference:" << std::endl;
  *x = 0;
  std::cout << &x << " " << x << " " << *x << std::endl;
  callByReference(x);

  free(x);

  return 0;
}

```

输出如下：
```bash
Call by pointer:
0x7fffd83b4190 0x7fffd0a87e70 0
0x7fffd83b4178 0x7fffd0a87e70 999

Call by reference:
0x7fffd83b4190 0x7fffd0a87e70 0
0x7fffd83b4190 0x7fffd0a87e70 888
```

由输出可以看出：当传入参数是指针时，指针地址（&x）变化了，说明进行了指针地址的复制（&x'）；当传入参数是引用时，指针地址没变，说明不需要进行复制。无论是哪种方式，指针指向对象的地址（x）一直不变。它们的关系如下图所示。

![](/assets/which-is-better-in-cpp-call-by-value-vs-reference-vs-pointer/pointer_vs_reference.png)


虽然在调用函数时速度差别不大，但作为函数返回值时有需要注意的地方：引用在定义时就被分配了指向的变量（即不能为空），且无法修改指向其他变量；指针可以为空，且可以修改指向其他变量。所以，如果要求函数返回值不能为空，可以考虑使用引用作为返回值，否则需要额外一步检查指针是否为空。

## 3. Value vs. Reference

- 当使用值（Value）传递时，本质上是Copy一份数据传入，这涉及到数据复制的过程，但函数内部可以直接读取该数据。

- 当使用引用（Reference）传递时，本质上是传一个地址，不涉及数据复制。但是在函数内部使用该数据时，需要额外一步读取操作（先读地址再读数据）。

在64位系统中，内存地址的大小为8 Bytes，在32位系统中，内存地址的大小为4 Bytes。而通常来说，C++中int类型大小为4 Bytes，double类型大小为8 bytes（当然基本类型大小在不同机器上可能不一样）。所以**比较两者的速度差别主要在于比较：数据复制耗时、指针读取耗时**。

一般来说，不考虑编译器优化、内存管理机制、CPU和缓存等影响的情况下，**如果数据的size很大（例如复杂结构体），采用引用速度更快**。引用传递总体来说会比值传递快和灵活，因为在大部分业界程序中，传递复杂对象的场景更多。

## 4. 总结与思考

对于Which is faster的问题，本文加粗字体部分就是结论。

但本文的题目是Which is better，其实我也给不出答案。程序语言每种机制的设计其作者都有相关的考虑，有的机制经过长时间的实践被证明是出色的，有的机制却造成业界不必要的困扰，饱受开发者吐槽。

即使是对同一种机制也可能存在正面和负面的声音，例如C语音的指针，这种机制使程序拥有优越的执行速度、出彩的传输性能，背后却是无尽的越界检查、空指针检查带来的痛苦。后续设计的程序语言都尽可能避免这种机制，或者说对其进行限制和封装。谷歌工程师表示，Chrome代码库中70%的严重漏洞与内存管理有关，许多工程师正在试图修复C/C++带来的内存管理问题，这带来的经济损失无可计量。

总而言之，编码时具体使用哪一种需要考虑到具体业务场景、对机器友好or对开发者友好等问题，没有绝对合适的方式。对程序语言感兴趣的可移步到推荐阅读。

## 参考与推荐阅读

[Which-is-faster-call-by-value-or-call-by-reference](https://www.quora.com/Which-is-faster-call-by-value-or-call-by-reference)  \\
[Chrome: 70% of all security bugs are memory safety issues](https://www.zdnet.com/article/chrome-70-of-all-security-bugs-are-memory-safety-issues/)  \\
[对开发者友好 or 对机器友好？](https://envoy.ink/blog/2018/07/19/developer-friendly-or-machine-friendly/)
