---
layout: post
title: "Exploring the Memory Management Strategy AllocSet in PostgreSQL MemoryContext"
date: 2025-07-27
categories: blog
permalink: /:categories/:year/:month/:title.html
---

## 0. 抽象与设计

MemoryContext 把内存分为不同层级和类别，方便管理和释放，整体以树状结构编排。它有如下特点：

与 C lib 的 malloc, free, realloc 的区别：

- palloc 在碰到 OOM 时不会抛出异常，而是以 elog(ERROR) 的方式退出；malloc 则返回 NULL 并且 errno 为 ENOMEM。因此在 PG 开发中从来不需要检查 palloc 返回的指针是否为 NULL。

- palloc(0) 不会返回 NULL，而是一个有效指针，强制分配的最小内存（后面会再提到），因此这个指针可以传给 repalloc 和 pfree；malloc(0) 会返回一个特殊 NULL，这个特殊 NULL 传给 pfree 不会报错。

-  pfree 和 repalloc 不能传入 NULL 作为参数，在代码中以 Assert 的形式检查并退出；free 传入 NULL 的行为是 do nothing，realloc 传入 NULL 等同于 malloc(size)。
    - 我理解，这么设计的原因首先是 PG 的内存管理建立在 MemoryContext 机制上，如果 repalloc 对 NULL 的行为与 remalloc 一致，会出现预期在 XxxMemoryContext 上 repalloc，结果把内存申请到 CurrentMemoryConetxt 上的情况，造成预期外的结果。

    - 其次是希望开发者清楚每一块内存的申请与回收，如果出现 pfree(NULL) 说明两种情况：一是写了 Bug 需要及时排查，二是用 pfree 做内存释放的兜底，不可取。

多种 MemoryContext 实现方式：

内存管理可以有很多种实现方式，只要继承 MemoryContext 并且实现函数表中对应函数即可：

```C
// src/include/utils/palloc.h
typedef struct MemoryContextData *MemoryContext;

// src/include/nodes/memnodes.h
typedef struct MemoryContextData
{
    // ...
    const MemoryContextMethods *methods;  // 函数表
    MemoryContext parent;                 // 
    MemoryContext firstchild;             // 同层级的首个 MemoryContext 指针
    MemoryContext prevchild;              // 同层级的前一个 MemoryContext 指针
    MemoryContext nextchild;              // 同层级的后一个 MemoryContext 指针
    // ...
    MemoryContextCallback *reset_cbs;     // 回调函数
} MemoryContextData;
```

MemoryContext 还有其他字段如`*reset_cbs`用于回调机制，在 Context 删除或重置时调用，不是本文的重点，这里就一笔带过。

函数表中对内存管理的函数进行了定义：

```C
// src/include/nodes/memnodes.h
typedef struct MemoryContextMethods
{
    // ...
    void *(*alloc) (MemoryContext context, Size size);  // palloc 函数
    void (*free_p) (void *pointer);                     // pfree 函数
    void *(*realloc) (void *pointer, Size size);        // repalloc 函数
    // ...
} MemoryContextMethods;
```

函数表中还有诸如`reset`, `delete_context`, `is_empty`等函数，本文不介绍也都省略。

PG 的 MemoryContext 标准实现是 AllocSet（src/backend/utils/mmgr/aset.c），下面介绍它的具体实现。

## 1. AllocSet 的基本结构

在 AllocSet 机制下，在需要时调用 malloc 向操作系统申请一大块内存称为 Block，当 PG 调用 palloc 时则在 Block 内部划分出 Chunk 分配出去。

以下是 AllocSet 的结构：
```C
// src/backend/utils/mmgr/aset.c
typedef struct AllocSetContext
{
    MemoryContextData header;                       // 继承 MemoryContext
    AllocBlock	blocks;                             // Blocks 的指针头部
    MemoryChunk *freelist[ALLOCSET_NUM_FREELISTS];  // Free chunk 列表
    // ...
} AllocSetContext;
typedef AllocSetContext *AllocSet;
```

由于 C 没有继承语法，实现继承的方法就是结构体头部放继承的对象，因此 MemoryContext 放最开头。
`blocks`是双向链表，管理目前已经申请的所有 Blocks。要划分 Chunk 都从链表头部 Block 里划分，分完了就申请新的 Block 继续放到链表头部。

```C
#define ALLOC_MINBITS          3   // Chunk 最小为 8 Bytes
#define ALLOCSET_NUM_FREELISTS 11
#define ALLOC_CHUNK_LIMIT      (1 << (ALLOCSET_NUM_FREELISTS-1+ALLOC_MINBITS))
```
`*freelist`管理当前空闲 Chunk 的指针数组，它的长度为 11。
这里通过宏定义也能看出 Chunk 至少为 8 Bytes，这也解释了为什么 palloc(0) 会返回一个有效指针，并且强制分配 8 Bytes 内存。也可以说只要申请内存小于 8 Bytes 都会分配足 8 Bytes。

宏定义`ALLOC_CHUNK_LIMIT`表示一个内存临界值，掐指一算这个临界值是 8192 Bytes，后面会用到它，这里先提前介绍：当申请的内存大于 8 KB 时则直接申请一个新的 Block，在上面进行 Chunk 的划分。

下面是 Block 的定义：

```C
// src/backend/utils/mmgr/aset.c
typedef struct AllocBlockData *AllocBlock;
typedef struct AllocBlockData
{
    AllocSet	aset;  // 持有该 Block 的 AllocSet 指针
    AllocBlock	prev;  // 节点的前一个 Block
    AllocBlock	next;  // 节点的后一个 Block
    char *freeptr;     // 指向当前 Block 内空闲内存的起始位置
    char *endptr;      // 指向当前 Block 的结束位置
} AllocBlockData;
```

由此可知，一个完全干净未分配任何 Chunk 的 Block 内存结构如下：
<img src="/assets/exploring-the-memory-management-strategy-allocset-in-postgresql-memorycontext/alloc_block_data.png" width = "345">

下面是 Chunk 的定义，只用了一个 unint64 来存储所有 Meta 信息：

```
// src/include/utils/memutils_memorychunk.h
typedef struct MemoryChunk
{
    /* bitfield for storing details about the chunk */
    uint64 hdrmask;
} MemoryChunk;
```

## 2. AllocSet 的内存分配实现

前面提到，当申请的内存大于 8 KB 时则直接申请一个新的 Block，以下就是申请并分配的核心过程（省略了一些内容）：
```C
// src/backend/utils/mmgr/aset.c
#define ALLOC_BLOCKHDRSZ	MAXALIGN(sizeof(AllocBlockData))
#define ALLOC_CHUNKHDRSZ	sizeof(MemoryChunk)
if (size > set->allocChunkLimit)
{
    // 计算需要 malloc 的大小：申请内存 + BlockData 元数据 + MemoryChunk 元数据
    blksize = chunk_size + ALLOC_BLOCKHDRSZ + ALLOC_CHUNKHDRSZ;
    block = (AllocBlock) malloc(blksize);

    // 由于整块内存都分配给 Chunk，因此 freeptr = endptr
    block->aset = set;
    block->freeptr = block->endptr = ((char *) block) + blksize;

    // MemoryChunk 就放在 BlockData 后面
    chunk = (MemoryChunk *) (((char *) block) + ALLOC_BLOCKHDRSZ);

    // 把申请的 blocks 放到链表第二个，简单的指针切换，不浪费篇幅
    // ...

    return MemoryChunkGetPointer(chunk);
}
```

可以看到这里申请的内存都分给一个 Chunk，因此不得不把这个 Block 放到链表的第二个，因为头部 Block 还有空闲内存（未划分 Chunk），后面流程中会用到它。

由此可知，一个划分了 Chunk 的 Block 内存结构如下：

<img src="/assets/exploring-the-memory-management-strategy-allocset-in-postgresql-memorycontext/alloc_block_data_with_multi_chunks.png" width = "400">
<img src="/assets/exploring-the-memory-management-strategy-allocset-in-postgresql-memorycontext/alloc_block_data_with_single_chunk.png" width = "180">

上图（左）这种结构可以叫做 Multi-chunk block，即一个 Block 被划分成了多个 Chunk。而上图（右）为大于 8 KB 的内存单独申请 Block 的情况，可以叫做 Single-chunk block。

如果申请的内存小于 8 KB 则先在`*freelist`中查找，如果刚好有合适大小的则直接返回：
```C
// src/backend/utils/mmgr/aset.c
typedef struct AllocFreeListLink
{
	MemoryChunk *next;
} AllocFreeListLink;

// 这块地址其实是空闲内存，闲着也是闲着，因此用来存储 AllocFreeListLink 结构
// 这个结构只有一个指针大小，而空闲内存至少 8 Bytes 因此一定放得下
#define GetFreeListLink(chkptr) \
	(AllocFreeListLink *) ((char *) (chkptr) + ALLOC_CHUNKHDRSZ)

// 获取 size 对应的链表头 index
fidx = AllocSetFreeIndex(size);
chunk = set->freelist[fidx];
if (chunk != NULL)
{
    // 通过该 Chunk 的地址拿下一个 Free chunk
    AllocFreeListLink *link = GetFreeListLink(chunk);

    // 把下一个 Free chunk 放到链表头
    set->freelist[fidx] = link->next;

    return MemoryChunkGetPointer(chunk);
}
```
这里 PG 做了一个叹为观止的操作，由于 Free chunk 是 Meta data + 空闲内存的结构，因此直接把下一个 Free chunk 的地址写在空闲内存中（空闲内存最小值是 8 Bytes，因此一定存得下指针），分配出去后再擦除，这样做就非常节省内存了，MemoryChunk 的结构可以做得非常简单。印象中以前这段代码还不是这样，当时 MemoryChunk 还叫做 AllocChunk，用一个 (void*) 多用途指针来存储 Next free chunk 地址。

如果 Free chunk 列表里没有合适大小的 Chunk，则从当前 Block 查看剩余空间是否足够。如果不足，说明当前 Block 剩余空闲内存连 8 KB 都没有，可以申请一个新的 Block 了。
在此之前需要把这部分小内存也放到 Free chunk 列表中，否则它就永远成为碎片，再也分配不出去了。这部分逻辑相对易懂，不展开描述。

创建新 Block 的逻辑和之前大同小异，也不重复地列举代码，只有一个重点：申请的 Block 的大小。每次申请 Block 的大小都是上一次的 Power 2，直到 maxBlockSize 或者 malloc 失败（系统内存不足）。malloc 失败则会执行右移操作（>>1）缩小申请的内存直到申请成功。

在 Block 上分配 Chunk 的逻辑就非常简单，仅需将 freeptr 移动：
```C
// 找到空闲内存起始位置
chunk = (MemoryChunk *) (block->freeptr);

// 移动空闲内存起始位置
block->freeptr += (chunk_size + ALLOC_CHUNKHDRSZ);
Assert(block->freeptr <= block->endptr);

return MemoryChunkGetPointer(chunk);
```

AllocSet 的整体结构如下图所示：

<img src="/assets/exploring-the-memory-management-strategy-allocset-in-postgresql-memorycontext/alloc_set.png">

1. `blocks`链表的头部是还有空闲内存（未划分 Chunk）的 Block，这也解释了为什么当申请内存 > 8 KB 时独立申请的 Block 要放在第二个，因为它是 Single-chunk block，已经划分完 Chunk 了。
2. `*freelist`管理所有之前申请又释放的 Chunks，这些内存并不是立刻归还给操作系统，而是先交由 AllocSet 维护，避免频繁的系统调用。

一次完整的内存申请流程如下：

<img src="/assets/exploring-the-memory-management-strategy-allocset-in-postgresql-memorycontext/palloc_procedure.png" width = "600">
