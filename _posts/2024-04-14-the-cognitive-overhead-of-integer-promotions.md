---
layout: post
title: "The Cognitive Overhead of Integer Promotions"
date: 2024-04-14
categories: blog
permalink: /:categories/:year/:month/:title.html
---

起因是碰到这样一个问题：
```
uint8_t a = 0x1;
请问 ~a 是多少？
A. 0xfe
B. 0xffff fffe
```
乍一看不就是选 A 吗？但答案却是 B，验证了一下确实是这样：
```C++
#include <stdint.h>
#include <stdio.h>

int main() {
  uint8_t a = 0x1;
  printf("%x, %zu\n", ~a, sizeof(~a));  // fffffffe, 4

  return 0;
}
```

仔细一想，模糊的记忆被唤起，是有在一些文章中看到过这个知识点：Integer promotions。


查阅资料看到：
> Integer promotion is the implicit conversion of a value of any integer type with rank less or equal to rank of int or of a bit-field of type _Bool(until C23)bool(since C23), int, signed int, unsigned int, to the value of type int or unsigned int.

简单来说就是小于 int 类型的被提升为 int 类型的过程。那么什么情况下会出现？

> Note: integer promotions are applied only
- as part of usual arithmetic conversions
- as part of default argument promotions
- to the operand of the unary arithmetic operators + and -
- to the operand of the unary bitwise operator ~
- to both operands of the shift operators << and >>

前两条就不用说了吧，太常见了：两个不同类型的值加减乘除，肯定会有一个类型提升；实参和形参类型对不齐也会类型提升。

文章开头的问题考的就是后面几个知识点，正负转换、取反和位移操作也会 Integer promotions！

可以通过汇编看一下这个过程：
```
movb    $1, -1(%rbp)
movzbl  -1(%rbp), %eax    // 挪到 eax 并在前面补 0
notl    %eax              // 取反
movl    %eax, %esi        // 结果挪到 esi
leaq    .LC0(%rip), %rdi
movl    $0, %eax
call    printf@PLT
movl    $0, %eax
leave
```
可以看到 0x1 被挪到 32-bit 的 eax 寄存器里，对整个 eax 寄存器取反，接着把整个 32-bit 结果挪到 esi 寄存器等着调用 printf 函数。

至于为什么要类型提升到 int 再操作，我猜和机器指令有关了，或许直接对 32-bit 的 eax 操作，比对 8-bit 的 al 更便宜。或许这就是 C++ 的心智负担。

所以为了避免踩到坑，尽量避免使用小于 int 的类型，如果碰到就做个类型转换。

## Ref

[Implicit conversions](https://en.cppreference.com/w/c/language/conversion)
