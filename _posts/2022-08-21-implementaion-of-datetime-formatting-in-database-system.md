---
layout: post
title: "Implementation of Datetime Formatting in Database system"
date: 2022-08-21
categories: blog
permalink: /:categories/:year/:month/:title.html
---

## 0. 前言

数据库系统对于日期类型一般支持格式化，例如 PostgreSQL 的 to_char 函数，ClickHouse 的 formatDatetime 函数，MySQL 的 to_char 函数。
大多数数据库对 Datetime 类型的存储和计算使用 int64_t，它能方便地表示任一时刻相对于 Unix epoch (i.e., 1970-01-01 00:00:00) 的秒数差。

要做格式化主要关注两个问题：
1. 如何快速将 Datetime 类型转成年月日分秒的概念；
2. 如何将 Format pattern 中的符号替换成转换好的年月日分秒。

## 1. ClickHouse (CK)

### 1.1 Lookup Table

- CK 的转换基于 Lookup table (LUT)，在堆内存中每个 Time zone 对应一个单例的 LUT，在 Runtime 实时初始化，为了节省内存，LUT 的一项为一天。

```C
// src/Common/DateLUT.h
/// This class provides lazy initialization and lookup of singleton DateLUTImpl objects for a given timezone.
class DateLUT : private boost::noncopyable
{
    // ...
    using DateLUTImplPtr = std::unique_ptr<DateLUTImpl>;
    /// Time zone name -> implementation.
    mutable std::unordered_map<std::string, DateLUTImplPtr> impls;
    // ...
}
```

```C
// src/Common/DateLUTImpl.h
#define DATE_LUT_MIN_YEAR 1925
#define DATE_LUT_MAX_YEAR 2283
#define DATE_LUT_YEARS (1 + DATE_LUT_MAX_YEAR - DATE_LUT_MIN_YEAR)

#define DATE_LUT_SIZE 0x20000

#define DATE_LUT_MAX (0xFFFFFFFFU - 86400)
#define DATE_LUT_MAX_DAY_NUM 0xFFFF
/// Max int value of Date32, DATE LUT cache size minus daynum_offset_epoch
#define DATE_LUT_MAX_EXTEND_DAY_NUM (DATE_LUT_SIZE - 16436)

class DateLUTImple
{
    // ...
    using Time = Int64;
    
    /// The order of fields matters for alignment and sizeof.
    struct Values
    {
        /// Time at beginning of the day.
        Time date;

        /// Properties of the day.
        UInt16 year;
        UInt8 month;
        UInt8 day_of_month;
        UInt8 day_of_week;
        UInt8 days_in_month;
    }
    
    static_assert(sizeof(Values) == 16);
    
    /// Offset to epoch in days (ExtendedDayNum) of the first day in LUT.
    static constexpr UInt32 daynum_offset_epoch = 16436;
    static_assert(daynum_offset_epoch == (1970 - DATE_LUT_MIN_YEAR) * 365 + (1970 - DATE_LUT_MIN_YEAR / 4 * 4) / 4);

    /// Lookup table is indexed by LUTIndex.
    Values lut[DATE_LUT_SIZE + 1];
    
    // ...
}
```

- CK 支持的时间范围为 1925-01-01 到 2283-01-01，约 130000 天，由于 LUT 是放在内存中的，所以不能够无限扩展时间，但是对于 OLAP 系统来说足够。
- Values 结构体记录 LUT 的一项内容，包括当天 0 点的时间戳、年月日和星期几等，在运行时可以直接获取，省下计算的时间。
- Values 结构体通过细致的排列保证内存对齐，每个结构体仅占用 16 Bytes 内存，因此每个 LUT 占用内存为 2 MB。
- 变量 daynum_offset_epoch 是 1925-01-01 和 1970-01-01 之间时间戳的差值，因为 Unix epoch 中 1970-01-01 为 0，小于该日期为负数，想要作为数组下标需要进行偏移，因此引入该变量。
- 通过上述结构体，获取时间的时分秒也是简单的，详见 toHour / toMinute / toSecond 函数。

### 1.2 基于函数指针调用的符号替换

- 根据 CK 官方文档的描述，formatPattern 函数的 Pattern 参数只能是常量，因此针对输入数据只允许按照一种 Pattern 进行格式化（这和 PostgreSQL 有一些区别）。
- CK 进行格式化主要有两个步骤：
  - 解析格式并生成模板，需要替换的部分用 0 代替，并存下对应的函数指针；
  - 循环每一行数据用同样的模板进行替换。
- 函数 parsePattern 如下：CK 用 % 加一个字母来表示不同格式（例如 %C 表示年，%d 表示日期），因此用 switch-case 即可。

```C
// src/Functions/formatDateTime.cpp
template <typename T>
String parsePattern(const String & pattern, std::vector<Action<T>> & instructions) const
{
    String result;
    // ...
    while (true)
    {
        const char * percent_pos = find_first_symbols<'%'>(pos, end);
        // ...
        switch (*pos)
        {
            // Year, divided by 100, zero-padded
            case 'C':
                instructions.emplace_back(&Action<T>::century, 2);
                result.append("00");
                break;
            // ...
        }
        ++pos;
        // ...
    }
    return result;
}
```

- 进行替换的函数如下：对于每一行数据，每个转换函数都调用一次（instruction.perform）。

```C
// src/Functions/formatDateTime.cpp
template <typename DataType>
ColumnPtr executeType(const ColumnsWithTypeAndName & arguments, const DataTypePtr &) const
{
    // ...
    for (size_t i = 0; i < vec.size(); ++i)
    {
        // ...
        for (auto & instruction: instructions)
            instruction.perform(pos, vec[i], time_zone);
    }
    // ...
    return col_res;
}
```


## 2. PostgreSQL (PG)

PG 函数 to_char 的入口是 timestamp_to_char 函数：

```C
// src/backend/utils/adt/formatting.c

Datum
timestamp_to_char(PG_FUNCTION_ARGS)
{
    Timestamp dt = PG_GETARG_TIMESTAMP(0);
    text *fmt = PG_GETARG_TEXT_PP(1), *res;
    TmToChar tmtc;
    struct pg_tm tt;
    struct fmt_tm *tm;
    // ...
    if (timestamp2tm(dt, NULL, &tt, &tmtcFsec(&tmtc), NULL, NULL) != 0)
        ereport(ERROR,
                (errcode(ERRCODE_DATETIME_VALUE_OUT_OF_RANGE),
                errmsg("timestamp out of range")));
    // ...
    if (!(res = datetime_to_char_body(&tmtc, fmt, false, PG_GET_COLLATION())))
        PG_RETURN_NULL();

    PG_RETURN_TEXT_P(res);
}
```
- timestamp2tm 函数将传入的参数（变量 dt）进行时间转换得到年月日时分秒等信息，存储到结构体中（变量 tmtc）。
- datetime_to_char_body 函数接受参数 Pattern（变量 fmt）和时间结构体（变量 tmtc）进行 Format。

### 2.1 淳朴的日期转换

- 实时转换，没有技巧。不展开讨论，j2date 和 dt2time 两个函数分别得到年月日和时分秒。

### 2.2 带有 Cache 的符号解析和替换

- PG 会将 Pattern 解析成 FormatNodes 结构体对象，FormatNodes 对象类似于 CK 里的函数指针，用于记录 Pattern 中哪个位置需要替换成什么格式，下面会举例说明。
- 由于 PG 支持每一行数据用不一样的 Pattern 进行 Format，对每一个数据都要解析一次 Pattern，因此 PG 采用 Cache 机制：

```C
// src/backend/utils/adt/formatting.c

/*
 * Format a date/time or interval into a string according to fmt.
 * We parse fmt into a list of FormatNodes.  This is then passed to DCH_to_char
 * for formatting.
 */
static text *
datetime_to_char_body(TmToChar *tmtc, text *fmt, bool is_interval, Oid collid)
{
    // ...
    if (fmt_len > DCH_CACHE_SIZE)
    {
        /*
         * Allocate new memory if format picture is bigger than static cache
         * and do not use cache (call parser always)
         */
        // ...
        parse_format(format, fmt_str, DCH_keywords, DCH_suff, DCH_index, DCH_FLAG, NULL);
    }
    else
    {
        /*
         * Use cache buffers
         */
        // ...
    }
    // ...
    return res;
}
```

- 如果这个 Pattern 的大小超过 128 Bytes，那么不把它放进 Cache 里，而是在线地把它解析成 FormatNodes，否则去 Cache 里查询。

```C
// src/backend/utils/adt/formatting.c

static void parse_format(FormatNode *node, const char *str, const KeyWord *kw,
                         const KeySuffix *suf, const int *index, uint32 flags, NUMDesc *Num);
```

- parse_format 函数的 7 个参数：
  - node 是用 palloc 申请好内存的指向 FormatNode 的指针，FormatNode 有3种类型：ACTION / CHAR / END，ACTION 表示要转换，CHAR 表示 char（即不是KeyWord），END表示结束。
  - str 是 to_char 传入进来的第二个参数，即 Pattern。
  - KeyWord 是描述格式（YYYY, MM, etc.）的结构体，*kw 传入进来了 PG 所有支持的结构体（Hard code 形式），结构体里有 5 个变量，描述名字、长度等等属性。
  - KeySuffix 是描述前缀（FM, th, etc.）的结构体，注意：这里虽然名字是前缀，但实际上有后缀，例如 th 其实是一种后缀。因此结构体里有一项属性叫 type，描述是 prefix 还是 postfix，KeySuffix 的内容不需要被替换。
  - index 是一个 int 数组，里面列了支持的 KeyWord 的在 *kw 中的初始位置；这个数组用在index_seq_search 函数中，用来快速过滤掉不支持的情况；数组的大小是 ASCI II 中 '～' 到 ' ' 之间的数量，显然是表示 PG 只支持这两个符号之间的符号，数组中的值如果是 -1 则表示不支持该符号，否则填上支持的 KeyWord 的 id（enum，也就是int），这里的实现非常巧妙，在下面的 index_seq_search 函数中详细介绍。
  - 其余参数不重要。
  
- parse_format 函数实现如下：本质上是顺序扫描 Pattern，构建 FormatNodes。例如 Pattern 为 "ABCYYYY-MM-DD123"，最终构建出 8 个 FormatNode：ABC / YYYY / - / MM / - / DD / 123 / END。

```C
static void
parse_format(FormatNode *node, const char *str, const KeyWord *kw,
             const KeySuffix *suf, const int *index, uint32 flags, NUMDesc *Num)
{
    FormatNode *n;
    n = node;
    while (*str)
    {
        int suffix = 0;
        const KeySuffix *s;
        
        // Parse **prefix** and skip them.
        
        if (*str && (n->key = index_seq_search(str, kw, index)) != NULL)
        {
            // A FormatNode = prefix + KeyWord + postfix
        }
        else if (*str)
        {
            /*
             * Process double-quoted literal string, if any
             */
        } // if-else
    } // while
    // The ending FormatNot
    n->type = NODE_TYPE_END;
    n->suffix = 0;
}
```

- index_seq_searcch 函数实现细节：

```C
/* ----------
 * Fast sequential search, use index for data selection which
 * go to seq. cycle (it is very fast for unwanted strings)
 * (can't be used binary search in format parsing)
 * ----------
 */
static const KeyWord *
index_seq_search(const char *str, const KeyWord *kw, const int *index)
{
    int poz;
 
     if (!KeyWord_INDEX_FILTER(*str))
         return NULL;
         
    if ((poz = *(index + (*str - ' '))) > -1)
    {
        const KeyWord *k = kw + poz;

        do
        {
            if (strncmp(str, k->name, k->len) == 0)
                return k;
            k++;
            if (!k->name)
                return NULL;
        } while (*str == *k->name);
    }
    return NULL;
}
```

- KeyWord_INDEX_FILTER 是非常简单的过滤，检查第一个字符是否支持，例如 '['、'z' 这种就可以直接过滤掉。如果支持，poz 是从 index 中获得的扫描的初始点，由于所有支持的 KeyWord 都被定义成 enum，也就是有序的，例如 B 开头那只有 DCH_B_C（5）、DCH_BC（6），index 在 'B' 位置为DCH_B_C（5），因此接下来的 while 循环中就会扫描 B 开头的几个 KeyWord（跳过 A 开头的），非常巧妙的实现。
- 这里可以停下来和 CK 的实现进行比较：
  - CK 直接跳过非 KeyWord，把内容直接贴到结果中；PG 会把它们解析成 FormatNode，带上 CHAR 的标签来表示它们不需要转换。因此本质上 CK 是做字符串替换，PG 是做字符串拼接。
  - PG 的格式表示是不定长的，使用 strncmp 来比较输入是否和 KeyWord 匹配，理论上复杂度为 O(m*n)，但是 PG 有搜索空间的优化。
- 最后将解析好的 FormatNodes 和时间信息传入 DCH_to_char 函数，循环 FormatNodes 进行字符串的拼接。


## 3. 总结
- 功能上来说，PG 支持的格式比 CK 的更多，因此 Format 的实现上 PG 比 CK 更复杂。
  - CK 只需要 Format 一次 Pattern，PG 需要多次 Format 因此引入 Cache 机制。
  - 本质上 CK 是做字符串替换，PG 是做字符串拼接。
- 特点上来说，CK 实时构建 Lookup table 加速日期查询，PG 则实时计算。
