---
layout: post
title: "Basics about Stack Trace and x86-64 for Machine-Level Programming"
date: 2024-07-28
categories: blog
permalink: /:categories/:year/:month/:title.html
---

## 0. 前言

做基础架构的开发，无论是性能优化还是问题排查，难免会和汇编打交道，在阅读了几篇相关文章后总结出一些关于 x86-64 架构、汇编以及内存堆栈的基础知识。

开始之前，先说明一些约定：
- 在 x86 架构下，内存通常采用小端序（Little Endian）。
- 栈的地址是从大到小延伸的，关于文中对栈地址的增减的描述，这里约定自然语言「往下移动」表示「往小地址移动」，即地址（值）减小。

## 1. 指令 push & pop

我们都知道 %rbp 的作用是保存当前调用函数栈的基地址，通常能在汇编代码中看到函数体的第一句是：`push %rbp`，最后一句是：`pop %rbp`。

因为在开启新的函数栈之前，%rbp 保存的是当前函数栈的基地址，为了保存新的函数栈基地址，需要把 %rbp 当前的值保存（push）到栈上，等到新函数执行完毕后再从栈恢复（pop）到寄存器中。

push 指令的用法：`push S`，效果是：

$$
\begin{align}
R[\%rsp] \leftarrow R[\%rsp] - 8  \\
M[R[\%rsp]] \leftarrow S
\end{align}
$$

第一行：%rsp 保存的地址往下移动 8 字节，给 %rbp 的值预留空间。

第二行：把 S 的内容放到 %rsp 指向的内存地址上，相当于放到内存栈上了。

pop 指令的用法是：`pop D`，效果是：

$$
\begin{align}
D \leftarrow M[R[\%rsp]] \\
R[\%rsp] \leftarrow R[\%rsp] + 8
\end{align}
$$

第一行：将 %rsp 指向的内容从内存复制到 D 上，通常为 %rbp。

第二行：将 %rsp 指向的地址往上移动 8 字节，回收内存栈空间。

## 2. 指令 call

call 指令顾名思义就是调用函数的意思，在跳入到函数体之前，它会将**这个函数返回后下一条指令的地址**保存到栈上。以下面的 C 语言程序为例：

```C
// basic_call.c
int func() {
    return 2;
}
int main() {
    return func();
}
```

使用 gcc 编译出二进制文件：

```bash
gcc basic_call.c -o basic_call
```

使用 gdb 执行文件并在 main 函数打断点：

```
(gdb) break main
Breakpoint 1 at 0x4004c1
(gdb) run
(gdb) disassemble
Dump of assembler code for function main:
   0x00000000004004bd <+0>:     push   %rbp
   0x00000000004004be <+1>:     mov    %rsp,%rbp
=> 0x00000000004004c1 <+4>:     mov    $0x0,%eax
   0x00000000004004c6 <+9>:     call   0x4004b2 <func>
   0x00000000004004cb <+14>:    pop    %rbp
   0x00000000004004cc <+15>:    ret
End of assembler dump.
```

使用`info register rbp`和`info register rsp`命令分别查看 %rbp 和 %rsp 的值：

```
(gdb) info register rbp
rbp            0x7fffffffb7b0      0x7fffffffb7b0
(gdb) info register rsp
rsp            0x7fffffffb7b0      0x7fffffffb7b0
```

使用`stepi`命令逐条执行汇编代码，直到执行 call：

```
(gdb) stepi
0x00000000004004b2 in func ()
(gdb) info register rsp
rsp            0x7fffffffb7a8      0x7fffffffb7a8
```

当执行到 func 函数时停下，查看 %rsp 的值，可以看到它变为 0x7fffffffb7a8，相比于之前的 0x7fffffffb7b0 往下移动了 8 字节，那么现在看这 8 字节里保存了什么内容：

```
(gdb) x/2g $rsp
0x7fffffffb7a8: 0x00000000004004cb      0x0000000000000000
```

注意到，这 8 字节保存的内容刚好是 call func 指令返回后的下一条指令的地址 0x00000000004004cb。接着执行函数体内容：

```
(gdb) stepi
0x00000000004004b3 in func ()
(gdb) disassemble
Dump of assembler code for function func:
   0x00000000004004b2 <+0>:     push   %rbp
=> 0x00000000004004b3 <+1>:     mov    %rsp,%rbp
   0x00000000004004b6 <+4>:     mov    $0x2,%eax
   0x00000000004004bb <+9>:     pop    %rbp
   0x00000000004004bc <+10>:    ret
(gdb) x/2g $rsp
0x7fffffffb7a0: 0x00007fffffffb7b0      0x00000000004004cb
```

可以看到函数体的第一行`push %rbp`执行完毕后，%rbp 之前保存的地址 0x00007fffffffb7b0 被保存到了栈上，符合上面关于 push & pop 的描述。

## 3. x86 vs. x86-64：寄存器

x86 架构原名为 IA32（Intel Architecture, 32-bit），本文仍然用 x86 表示 IA32。当前最广泛使用的 x86-64 架构最早是 AMD 公司先推出的，在此之前 Intel 尝试过推出 64 位的处理器，名为安腾（Itanium）系列，但它不与 x86 兼容这一点使得这款处理器很难推广开来，最终在 2001 年停产。AMD 从中看到了机会，在 2002 年推出了与 x86 兼容的 x86-64 系列处理器，得益于兼容性，这款处理器帮助 AMD 获得了不少高端服务器的市场。Intel 此时发现，做一款向前兼容的处理器貌似才是正路，于是在 2004 年也推出了与 x86 兼容的奔腾 4 至强（Pentium 4 Xeon）处理器。从此以后，向前兼容的 x86-64 正式广泛推广开来。

x86-64 相比于 x86 寄存器最大的变化是从 8 个寄存器增多为 16 个寄存器，并且每个寄存器的大小从 32 位提升到 64 位。

![](/assets/basics-about-stack-trace-and-x86-64-for-machine-level-programming/x86_64_register.png)

这张图（来源于 Ref）展示了 x86-64 所有寄存器，x86 原本的 8 个寄存器为：%eax，%ebx，%ecx，%edx，%esi，%edi，%ebp，%esp。x86-64 将它们保留并扩展到 64 位，名字中的“e”改为“r”，即 %rax，%rbx，%rcx，%rdx，%rsi，%rdi，%rbp，%rsp。以 %rax 为例，你仍然能通过 %eax 去读取它的低 32 位，也能通过 %ax 去读取它的低 16 位，其他寄存器类似。

x86-64 还额外引入了 %r8 - %r15 共 8 个 64 位寄存器，同样可以直接读取它们的低 32 位、低 16 位和低 8 位。

注意到途中每个寄存器右边都有一些文字描述这个寄存器的具体作用，例如最为大家熟知的，%rax 用来保存函数的返回值，%rsp 用来保存函数栈指针，其他寄存器的作用会在后文中有介绍。

## 4. x86 vs. x86-64：传参

### 4.1 x86 传参

在 x86 架构下调用函数，函数的参数都是保存在栈上的，参数由右往左依次压入栈内，接着压入函数返回后的下一条指令的地址，最后压入 %ebp，此时 %ebp 往小地址就是调用函数的栈内容，%ebp 往大地址就是函数的参数。以下面的 C 语言程序为例：

```C
// basic_1.c
int add(int a, int b) {
    return a + b;
}
int main() {
    return add(666, 888);
}
```

使用 gcc 编译出汇编代码：

```bash
gcc -m32 -S basic_1.c -o basic_1_32.s
```

得到汇编代码如下（为了简化展示，手动删除了一些无关代码）：

```
add:
  pushl %ebp
  movl  %esp, %ebp
  movl  8(%ebp), %edx
  movl  12(%ebp), %eax
  addl  %edx, %eax
  popl  %ebp
  ret
main:
  pushl %ebp
  movl  %esp, %ebp
  pushl $888
  pushl $666
  call  add
  addl  $8, %esp
  leave
  ret
```

可以看到在 call add 语句之前依次执行了`push $888`和`push $666`操作，这里就是将函数的参数 888 和 666 依次压入栈中，接着在调用 add 函数后的第一句为`push %ebp`将 %ebp 保存在栈上。可以看到 add 函数体中通过`8(%ebp)`和`12(%ebp)`偏移寻址分别读取两个参数的值，最终将相加得到的结果放到 %eax 中作为返回值。

C 语言中的 printf 也是依靠这个机制实现的，其实就是根据要读取的类型计算出偏移的大小，例如 %ld, %d，要读取 %d 的对应的参数值，就需要跨过 %ebp（4 字节），返回地址（4 字节），long 类型（8 字节），一共 16 字节，即读取`16(%ebp)`。

读到这里，可以停下来思考以下几种场景：

1. 在函数体中定义 char *c，访问`(char *)&c - N`以及`(char *)&c + N`，其中 N >= 1，会发生什么？显然 c 是分配在栈上的，如果访问 - N 的内容则是在访问本函数栈上的内存，当然也可能越过 %esp，超过栈空间就会 segfault；如果访问 + N 的内容，理论上是在回溯之前函数栈上的内容。

2. 尝试修改 char *s = "abcd"。这样一段字符不是放在栈上的，而是放在 text 只读区，如果尝试修改 s[0] = 'A'，内核发现程序尝试修改只读内容，就会将程序 kill 掉。

### 4.2 x86-64 传参

x86-64 对函数传参做了优化，前 6 个参数会分别保存在 %rdi，%rsi，%rdx，%rcx，%r8，%r9 中，后面的参数则和 x86 一样保存在栈上。

对同样一段代码使用 64 位编译出汇编代码：

```bash
gcc -m64 -S basic_1.c -o basic_1_64.s
```

得到汇编代码如下（为了简化展示，手动删除了一些无关代码）：

```
add:
  pushq %rbp
  movq  %rsp, %rbp
  movl  %edi, -4(%rbp)
  movl  %esi, -8(%rbp)
  movl  -4(%rbp), %edx
  movl  -8(%rbp), %eax
  addl  %edx, %eax
  popq  %rbp
  ret
main:
  pushq %rbp
  movq  %rsp, %rbp
  movl  $888, %esi
  movl  $666, %edi
  call  add
  popq  %rbp
  ret
```

可以看到在 main 函数体中将参数 666 和 888 分别放到了 %edi 和 %esi 中，add 函数体则直接使用它们。

以下面 C 语言程序为例：

```C
int add(int a, int b, int c, int d, int e, int f, int g, int h) {
    return a + b + c + d + e + f + g + h;
}
int main() {
    return add(666, 888, 1, 2, 3, 4, 5, 6);
}
```

得到汇编代码如下（为了简化展示，手动删除了一些无关代码）：

```
main:
  pushq %rbp
  movq  %rsp, %rbp
  pushq $6
  pushq $5
  movl  $4, %r9d
  movl  $3, %r8d
  movl  $2, %ecx
  movl  $1, %edx
  movl  $888, %esi
  movl  $666, %edi
  call  add
  addq  $16, %rsp
  leave
  ret
```

可以看到前 6 个参数是按照规则放到寄存器中的，而第 7 和第 8 个参数和 x86 一样按照从右往左的顺序 push 到函数栈上。

## 5. x86-64 的 Conditional move

x86 还有 cmov 系列指令（以 cmov 开头，后缀根据不同的数据类型变化）。根据名字直接翻译为：条件移动指令，效果是：如果满足条件则执行移动操作。

我们都知道处理器对 if-else 语句有分支预测（Branch prediction）的能力，如果碰到预测失败（Branch misprediction）会浪费掉好几十个时钟周期，因此处理器中对程序有这样的优化：干脆将 if 和 else 两个分支的结果都计算出来，最后才判断用哪个，这样既能高效完成 if-else 分支，又不会打破处理器流水线执行的形式。这个优化就依靠 cmov 系列指令。

考虑如下代码：

```C
// basic_cm.c
int myfunc(int a, int b) {
    if (a >= 0) {
        return a;
    }
    return a + b;
}
int main() {
    return myfunc(1, 2);
}
```

对上面的代码使用`-O2`参数编译：

```bash
gcc -m64 -O2 -S basic_cm.c -o basic_cm.s
```

得到汇编代码如下（为了简化展示，手动删除了一些无关代码）：

```
myfunc:
  addl  %edi, %esi  ; 将 a + b 相加并保存到 %esi
  movl  %edi, %eax  ; 将 a 保存到 %eax
  testl %edi, %edi  ; if (a >= 0)
  cmovs %esi, %eax  ; 将原本计算好的 a + b 保存到 %eax
  ret
```

可以看到 myfunc 函数体连 %rbp 和 %rsp 的处理都没有了，这是因为函数足够简单，编译器将其优化成这样了，省去了对寄存器的保存和恢复的几个时钟周期。

其中前两行相当于把 if 和 else 两个分支的结果都计算完成，一个放在 %esi 上，一个放在 %eax 上，等待判断使用哪一个结果，使用 cmovs 指令将结果替换。

上面的汇编代码里没有 jmp 指令，完全按照流水线形式执行，也就避免了分支预测。但读者肯定也看出这种优化适用的场景限制很大，首先 if-else 两个分支必须足够简单，否则编译器可能无法优化成 Conditional move 的形式，也可能两条分支都计算一遍计算量太重，反而变成负优化。其次如果两条分支接的是逻辑语句，那么同样不适用。

## 6. x86-64 的 Callee-save 寄存器

Callee-save 寄存器表示寄存器上的值由被调用者（Callee）负责保存，返回之前必须恢复原状，调用者（Caller）能够接着使用；Caller-save 寄存器表示寄存器上的值由调用者负责保存，被调用者可以放心覆盖寄存器的值。例如 A 函数调用 B 函数，那么 A 函数就是 Caller，B 函数就是 Callee。其中 %rbp 算是 Callee-save 寄存器，因为函数使用前都要保存它；%rax 算是 Caller-save 寄存器，函数可以直接将返回值往上放。

考虑如下代码：

```C
// basic_fact.c
int fact(int x) {
    if (x <= 0) {
        return 1;
    }
    int x_1 = x - 1;
    return x * fact(x_1);
}
int main() {
    return fact(6);
}
```

对上面代码使用`-O1`参数编译，得到汇编代码如下（为了简化展示，手动删除了一些无关代码）：

```
fact:
  movl  $1, %eax        ; 将 1 放到返回值中
  testl %edi, %edi      ; if (x <= 0)
  jle   .L5             ; return 1
  pushq %rbx            ; 将 %rbx 的值保存到栈上
  movl  %edi, %ebx      ; 将参数保存到 %ebx 上
  leal  -1(%rdi), %edi  ; x_1 = x - 1
  call  fact            ; fact(x_1)
  imull %ebx, %eax      ; x * (%rax)
  popq  %rbx            ; 从栈上恢复 %rbx 的值
  ret
 .L5:
  ret
```

这段汇编代码中，fact 函数体使用 %ebx 来保存入参的值，并最终通过`imull %ebx, %eax`来计算结果。而作为 Callee-save 的寄存器 %rbx，使用之前需要先保存到栈上，函数返回前再恢复出来。

## 7. x86-64 的 Red zone

x86-64 还有一个特性：在一些特殊情况下，%rsp 可以不移动，并且能使用往下地址里“未分配”的空间，以此节省掉移动 %rbp 和 %rsp 的时钟周期，这部分能够使用的“未分配”空间称为 Red zone。

以如下 C 语言代码为例：

```C
// basic_rsp.c
int red_zone(int i) {
    int a[4] = {1, 2, 3, 4};
    int idx = i & 4;
    return a[idx];
}
int main() {
    return red_zone(3);
}
```

对上面代码使用`-O1`参数编译，得到汇编代码如下（为了简化展示，手动删除了一些无关代码）：

```
red_zone:
  movl   $1, -16(%rsp)
  movl   $2, -12(%rsp)
  movl   $3, -8(%rsp)
  movl   $4, -4(%rsp)
  andl   $4, %edi
  movslq %edi, %rdi
  movl   -16(%rsp,%rdi,4), %eax
  ret
```

按照正常流程来说，int a[4] 需要在栈上分配 16 字节的内容，但通过汇编可以看出，%rsp 直接往“未分配”的栈空间里放入 4 个 int 的值，整个过程只有 %rsp 的偏移寻址，没有 %rsp 的移动。官方称整个区域为 Red zone，它最大能使用 128 字节的空间。

但是在我实际验证中，貌似最多用到 120 字节，大于等于这个数量时，%rsp 就会开始移动：

```C
// basic_rsp_128.c
long red_zone(int i) {
    long a[15] = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15};
    int idx = i & 15;
    return a[idx];
}
int main() {
    return red_zone(3)
}
```

```
red_zone:
  subq  $8, %rsp
  movq  $1, -120(%rsp)
  movq  $2, -112(%rsp)
  movq  $3, -104(%rsp)
  movq  $4, -96(%rsp)
  movq  $5, -88(%rsp)
  movq  $6, -80(%rsp)
  movq  $7, -72(%rsp)
  movq  $8, -64(%rsp)
  movq  $9, -56(%rsp)
  movq  $10, -48(%rsp)
  movq  $11, -40(%rsp)
  movq  $12, -32(%rsp)
  movq  $13, -24(%rsp)
  movq  $14, -16(%rsp)
  movq  $15, -8(%rsp)
  andl  $15, %edi
  movq  -120(%rsp,%rdi,8), %rax
  addq  $8, %rsp
  ret
```
尽管如此，也仅是 %rsp 发生了加减，不涉及 %rbp 的保存恢复。

## 8. XMM 系列寄存器

x86-64 的浮点数有专门的 XMM 系列寄存器 %xmm0 - %xmm15，一共 16 个，每个都是 128-bit，即 8 字节。返回值放在 %xmm0 中，在浮点数的指令里，不存在内存到内存的操作，Destination 都是寄存器，Source 是寄存器或内存。注意，Source 连立即数都不能是。

全部的 xmm 寄存器都是 Caller-save 的，函数可以放心使用。参数也是放在 xmm 里的，第 1 个 float 参数放在 %xmm0 中，第 2 个放在 %xmm1 中，以此类推最多放到 %xmm7，再往后的参数就放在 %rdi 中。如果参数中穿插了一些其他类型，则遵循 x86-64 基本的函数调用规则，例如 func(float *a, float b, int c)，函数参数分别放在 %rdi，%xmm0 和 %esi 中。

## 最后

本文的例子主要来自《x86-64 Machine-Level Programming》这篇文章，文章里面还讲了 Data types，Loop control 和 Procedure，感兴趣的话可以去阅读。

## Ref

《x86-64 Machine-Level Programming》by Randal E. Bryant and David R. O'Hallaron.（文中图片来源）
