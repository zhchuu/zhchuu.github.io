---
layout: post
title: "Implementation of Decimal / Numeric Type in Database System"
date: 2022-07-16
categories: blog
permalink: /:categories/:year/:month/:title.html
---

## 0. 前言

Decimal 类型也叫 Numeric 类型，是现代数据库必备的类型之一，它用来表示高精度的数字以及做高精度运算。
本文简单介绍三种现代数据库的 Decimal / Numeric 类型的实现。

## 1. ClickHouse (CK)

### 1.1 用法

- Decimal(P, S)，其中 P 表示 Precision，S 表示 Scale。P 的取值范围为 [1, 76]，S 的取值范围为 [0, P]。
- 根据 P 的取值不同，Decimal 的定义也可以写成：
  - P 取值在 [1, 9]：Decimal32(S)
  - P 取值在 [10, 18]：Decimal64(S)
  - P 取值在 [19, 38]：Decimal128(S)
  - P 取值在 [39, 76]：Decimal256(S)

### 1.2 实现思想

- CK 对 Decimal 的实现是把它看作有符号整型，因此 CK 所支持的 Decimal 是有精度范围限制的。

```C
// Source code: base/base/Decimal.h

using Decimal32 = Decimal<Int32>;
using Decimal64 = Decimal<Int64>;
using Decimal128 = Decimal<Int128>;
using Decimal256 = Decimal<Int256>;

template <typename T>
struct Decimal
{
    // ...
    T value;
}
```

### 1.3 实现

#### 1.3.1 Comparison

```C
// Source code: base/base/Decimal.h

static bool compare(A a, B b, UInt32 scale_a, UInt32 scale_b)
{
    static const UInt32 max_scale = DecimalUtils::max_precision<Decimal256>;
    if (scale_a > max_scale || scale_b > max_scale)
        throw Exception("Bad scale of decimal field", ErrorCodes::DECIMAL_OVERFLOW);
                        
    Shift shift;
    if (scale_a < scale_b)
        shift.a = static_cast<CompareInt>(DecimalUtils::scaleMultiplier<B>(scale_b - scale_a));
    if (scale_a > scale_b)
        shift.b = static_cast<CompareInt>(DecimalUtils::scaleMultiplier<A>(scale_a - scale_b));

    return applyWithScale(a, b, shift);
}
```

在进行 Decimal 的比较时，需要传入两个值的 Scale 用于对齐小数点。例如：123.4 (p=4, s=1) vs. 123.456 (p=6, s=3)，对应有符号整型（int32_t）为 1234 vs. 123456，
前者 Scale 为 1 后者为 3，因此需要对前者进行 Rescale，得到的比较为 123400 vs. 123456。

#### 1.3.2 精度推导

- CK 的精度推导是简单粗暴的，牺牲了计算的精确性，换来实现的简便性。
- 两个 Decimal 之间进行任何操作，结果的 Precision 都取两者中最大的；
Scale 的规则也简单好理解，乘法则将两个 Scale 相加，除法则使用被除数的 Scale。这些规则在官方文档都有说明。

```C
// Source code: src/Core/DecimalFunctions.h

template <bool is_multiply, bool is_division, typename T, typename U, template <typename> typename DecimalType>
inline auto binaryOpResult(const DecimalType<T> & tx, const DecimalType<U> & ty)
{
    UInt32 scale{};
    if constexpr (is_multiply)
        scale = tx.getScale() + ty.getScale();
    else if constexpr (is_division)
        scale = tx.getScale();
    else
        scale = (tx.getScale() > ty.getScale() ? tx.getScale() : ty.getScale());

    if constexpr (sizeof(T) < sizeof(U))
        return DataTypeDecimalTrait<U>(DecimalUtils::max_precision<U>, scale);
    else
        return DataTypeDecimalTrait<T>(DecimalUtils::max_precision<T>, scale);
}

```

#### 1.3.3 Calculation

- 加减运算需要两个操作数小数点对齐，因此参考 Comparison 中 Rescale 的方法，当两个 Decimal 对齐后直接相加即可。
- 乘法运算不需要小数点对齐，直接相乘即可。
- 除法运算需要对被除数 Rescale，例如：5.000 (p=4, s=3) / 3.0 (p=2, s=1)，两者对应的有符号整型为 5000 和 30。
根据上面推导的规则，结果的 Scale = 3，5000 / 3 = 1666，再加上小数点（Scale = 3）即可得到 1.666。

### 1.4 优缺点

- 优点：用 C++ 原生的 int32_t / int64_t 等整型类型进行运算速度快；精度推导简单粗暴，实现简单，计算效率高。
- 缺点：支持的精度范围有限，不能无限扩展；精度推导规则太简单，一旦精度超过限制则直接报错，不能根据结果动态扩展或调整精度。


## 2. MySQL

### 2.1 用法

- Decimal(P, S)，其中 P 表示 Precision，S 表示 Scale。P 的取值范围为 [1, 65]，S 的取值范围为 [0, 30]。

### 2.2 实现思想

- 为了支持更多的精度推导规则和方便地扩展精度，MySQL 的设计更为复杂。首先，不再用 C++ 原生的类型来表示 Decimal，而是自定义能够自解释的结构体来表示 Decimal。

```C
// Source code: include/decimal.h

typedef int32 decimal_digit_t;

struct decimal_t {
    int intg, frac, len;
    bool sign;
    decimal_digit_t *buf;
};
```

- 变量含义：
    - intg 和 frac 分别表示整数部分和小数部分的长度；sign 表示符号（true 为负数）。
    - buf 是 int32 数组，len 是该数组的长度。
    - buf 用来存储 Decimal 的数字，一个 int32 最多存储 9 个数字，理论上这种结构能够存储无限长度的 Decimal，只需要不断扩展 buf 数组长度即可。

- 举例：12345.6789 (p=9, s=4) 存储为：
```C
struct decimal_t {
    int intg = 5, frac = 4, len = 2;
    bool sign = false;
    decimal_digit_t *buf = {12345, 678900000};
}
```

- 举例：1234567890123.456789 (p=19, s=6) 存储为：
```C
struct decimal_t {
    int intg = 13, frac = 6, len = 3;
    bool sign = false;
    decimal_digit_t *buf = {123456789, 12300000, 456789000};
}
```

### 2.3 实现

MySQL 的 Decimal 相关实现都在 include/decimal.h 和 strings/decimal.cc 中。

#### 2.3.1 Comparison

```C
// Source code: strings/decimal.cc

int decimal_cmp(const decimal_t *from1, const decimal_t *from2) {
  if (likely(from1->sign == from2->sign)) return do_sub(from1, from2, nullptr);
  
  // Reject negative zero, cfr. string2decimal()
  assert(!(decimal_is_zero(from1) && from1->sign));
  assert(!(decimal_is_zero(from2) && from2->sign));

    return from1->sign > from2->sign ? -1 : 1;
}
```
Decimal 的比较逻辑很简单，如果两者符号不同则直接返回结果；否则相减看结果。

#### 2.3.2 精度推导

对于：a op b

- 加减：
```
S = max(a.S, b.S)
P = max(a.P, b.P)
```

- 乘法：
```
S = a.S + b.S
P = a.P + b.P
```

- 除法：
```
S = frac = a.S + b.S
intg = (a.P - a.S) - (b.P - b.S) + 1
P = intg + frac
```

#### 2.3.3 Calculation

- 加减：在小数点对齐后进行如下操作，
  - 步骤一：将两数中 Scale 较大的数的末尾部分拷贝到结果中（如下的 $$y_5, y_6$$）；
  - 步骤二：将中间对齐部分进行相加（如下的$$x_4 . x_5 x_6 x_7 + y_1 . y_2 y_3 y_4$$）；
  - 步骤三：将两数中整型较大的数多出来的部分拷贝到结果中（如下的$$x_1, x_2, x_3$$）。

$$
\begin{aligned}
x_1 \ x_2 \ x_3 \ | \ x_4 &. x_5 \ x_6 \ x_7 
 \ | \\
 | \  y_1 &. y_2 \ y_3 \ y_4 \ \ | \ y_5 \ y_6
\end{aligned}
$$

```C
// Source code: strings/decimal.cc

/* part 1 - max(frac) ... min (frac) */
while (buf1 > stop) *--buf0 = *--buf1;
                                         
/* part 2 - min(frac) ... min(intg) */
carry = 0;
while (buf1 > stop2) {
    ADD(*--buf0, *--buf1, *--buf2, carry);
}
                                                     
/* part 3 - min(intg) ... max(intg) */
buf1 = intg1 > intg2 ? ((stop = from1->buf) + intg1 - intg2)
    : ((stop = from2->buf) + intg2 - intg1);
while (buf1 > stop) {
    ADD(*--buf0, *--buf1, 0, carry);
}
```

- 乘法：乘法的关键点如下，
  - 运算之前判断相乘后的结果是否超过最大精度限制，超过精度有两种情况：
    - 小数部分让出精度也不足弥补溢出的数量，精度严重损失，返回 E_DEC_OVERFLOW 错误；
    - 小数部分让出精度弥补溢出，精度少量损失，结果仍可用，返回 E_DEC_TRUNCATED 错误；
  - 引出 int64_t 的中间变量 dec2 来存储乘法结果，因为两个 int32_t 相乘可能溢出；
  - 用两个 for 循环来将两个 Decimal 的 buf 相乘，两个 buf 数组中的 int32_t 相乘的结果用 int64_t 存储，
  高 32 位为进位，低 32 位为当前结果。

```C
// Source code: strings/decimal.cc

using dec1 = decimal_digit_t;
/// A wider variant of dec1, to avoid overflow in intermediate results.
using dec2 = int64_t;

for (buf1 += frac1 - 1; buf1 >= stop1; buf1--, start0--) {
    carry = 0;
    for (buf0 = start0, buf2 = start2; buf2 >= stop2; buf2--, buf0--) {
        dec1 hi, lo;
        dec2 p = ((dec2)*buf1) * ((dec2)*buf2);
        hi = (dec1)(p / DIG_BASE);
        lo = (dec1)(p - ((dec2)hi) * DIG_BASE);
        ADD2(*buf0, *buf0, lo, carry);
        carry += hi;
    }
    if (carry) {
        if (buf0 < to->buf) return E_DEC_OVERFLOW;
        ADD2(*buf0, *buf0, 0, carry);
    }
    for (buf0--; carry; buf0--) {
        if (buf0 < to->buf) return E_DEC_OVERFLOW;
        ADD(*buf0, *buf0, 0, carry);
    }
}
```

- 除法：除法参考自 Donald Knuth 在 The Art of Computer Programming (TAOCP) 中描述的大整数运算，这里不展开描述和解释。

### 2.4 优缺点

- 优点：理论上可以支持比 CK 更长的精度（CK 原本只支持最大精度为 38，近几年才扩展到 76）；支持小数点自适应压缩，尽可能保证运算完成；
- 缺点：Decimal 结构体的维护复杂程度更高，涉及 Truncate 运算数、计算挤压小数空间等过程，效率相对更低。


## 3. PostgreSQL (PG)

### 3.1 实现思路

PG 的设计比 MySQL 更加复杂，首先它用两种结构存储 Decimal：一种为 NumericData 用于存储；一种为 NumericVar 用于内存中计算。
最大 Precision 为 1000，这里只对 NumericVar 进行简单介绍。

```C
// Source code: src/backend/utils/adt/numeric.c

#define NBASE            10000
#define HALF_NBASE       5000
#define DEC_DIGITS       4  /* decimal digits per NBASE digit */
#define MUL_GUARD_DIGITS 2  /* these are measured in NBASE digits */
#define DIV_GUARD_DIGITS 4
```

- 基本实现思路和 MySQL 有相似之处，都使用数组来存储 Decimal 中的数字。
- 在使用数组存储 Decimal 中的数字过程中，每个数组元素的范围是 [0, NBASE - 1]，PG 选择 10000 有如下必要条件：
  - NBASE * NBASE < INT_MAX，可以预留一些空间来处理进位；
  - 方便 I/O 转换和 Rounding 操作；
  - 一些历史原因。

总的来说这些条件并不是充分必要的，只是在各种历史原因之下选择了这个值。

```C
// Source code: src/backend/utils/adt/numeric.c

typedef int16 NumericDigit;

typedef struct NumericVar
{
    int ndigits;           /* # of digits in digits[] - can be 0! */
    int weight;            /* weight of first digit */
    int sign;              /* NUMERIC_POS, _NEG, _NAN, _PINF, or _NINF */
    int dscale;            /* display scale */
    NumericDigit *buf;     /* start of palloc'd space for digits[] */
    NumericDigit *digits;  /* base-NBASE digits */
} NumericVar;
```

- 内存计算的结构：
  - digits 为柔性数组，存储 Decimal 中的数字，单位是 int16（相比 MySQL 的 int32 小），放下 NBASE 绰绰有余；
  - ndigits 为 digits 数组长度；
  - weight 表示 digits 数组的第一个元素（int16）基于 NBASE 的权重，根据代码注释中的描述：
  Note: the first digit of a NumericVar's value is assumed to be multiplied by NBASE ** weight. 下面有例子说明；
  - sign 顾名思义，表示符号；
  - dscale 表示 Decimal 的 Scale，PG 是每一行 Decimal 都有各自精度，因此需要有一个变量来描述每一个 Decimal 的精度；
  - buf 为 digits 数组之前一小段内容，为了方便理解可以先不看。
  
- 举例：0
```C
typedef struct NumericVar
{
    int ndigits = 0;
    int weight = 0;
    int sign = NUMERIC_POS;
    int dscale = 0;
    NumericDigit *buf = NULL;
    NumericDigit *digits = {0};
} NumericVar;
```

- 举例：0.5
```C
typedef struct NumericVar
{
    int ndigits = 1;
    int weight = -1;
    int sign = NUMERIC_POS;
    int dscale = 1;
    NumericDigit *buf = NULL;
    NumericDigit *digits = {5000};
} NumericVar;
```
这里 weight 为 -1 是因为：5000 * NBASE^(-1) = 5000 * 1/10000 = 0.5

- 举例：0.9
```C
typedef struct NumericVar
{
    int ndigits = 1;
    int weight = -1;
    int sign = NUMERIC_POS;
    int dscale = 1;
    NumericDigit *buf = NULL;
    NumericDigit *digits = {9000};
} NumericVar;
```

- 举例：1.1
```C
typedef struct NumericVar
{
    int ndigits = 2;
    int weight = 0;
    int sign = NUMERIC_POS;
    int dscale = 1;
    NumericDigit *buf = NULL;
    NumericDigit *digits = {1, 1000};
} NumericVar;
```
这里 weight 为 0 是因为 digits 第一个元素为 1，而 1 * NBASE^(0) = 1。

### 3.2 实现

加减乘除等运算的实现和 MySQL 类似，在此不展开描述和解释。

### 3.3 优缺点
- 优点：支持精度非常高，绝大多数领域（包括天文数据、卫星数据等）都够用。
- 缺点：复杂程度高，维护和计算效率低。
