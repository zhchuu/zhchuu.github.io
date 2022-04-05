---
layout: post
title: "Understanding Roaring Bitmaps"
date: 2022-04-05
categories: blog
permalink: /:categories/:year/:month/:title.html
---

## About bitmaps
- Bitmaps 数据结构可以表示集合，因此它的使用场景必须是数据不重复的。
- Hash 也可以用来表示集合，相比而言 bitmaps 的好处是方便地进行集合之间交并差操作。
- Bitmaps 的本质是用 bit 的0和1来表示元素是否存在，一个 int32 有32个比特位，能够表示32个元素（0 - 31）；因此，一个集合中最大的元素是$$N$$，就要申请一个包含$$N/32$$个 int32 的数组，或者包含$$N/64$$个 int64 的数组。
- 但是如果集合的元素很稀疏，会导致申请大量为0的 int32 或 int64，因此衍生了一些压缩算法。

## A typical bitmaps compression algorithm
- 经典的压缩算法有：
  - Concise（[Compressed N Composable Integer Set](https://arxiv.org/abs/1004.0403)）
  - WAH（[Word Aligned Hybrid](https://sdm.lbl.gov/~kewu/ps/LBNL-49626-tods.pdf)）
  - EWAH（[Enhanced Word Aligned Hybrid](https://arxiv.org/abs/0901.3751)）
  - RBM（[Roaring bitmaps](https://arxiv.org/pdf/1402.6407.pdf)）
- About EWAH
  - 数组元素为 int64 (即long)，论文中描述为 Word。
  - 有两种 Word：
    - 一种存储 Data （Literal Word, LW），即存储真实元素。
    - 另一种当作"路标"（Running Length Word, RLW），是压缩的重要一环，这种 Word 存储两个信息：1. 当前 Skip 掉了多少全部为0的 Word；2. 后面有多少连续的 Data 类型的 Word。
  - 举个例子：集合中包含元素0，2和1000，如果不使用任何压缩算法则需要申请一个有$$floor(1000/64)+1$$个（16个）Word 的数组，第一个 Word 为（0000 ... 0101），
  最后一个 Word 为（0000 ...0001 ... 0000），在第41个位置标记1；中间有整整14个 Word 都"浪费"了。
  - 如果使用 EWAH，只需要申请3个Word，第一个和第三个如上所示，第二个则记录中间 Skip 了14和 Word，并且右边有连续1个 Word。
  - 上述例子只是很简单地阐述算法思想，实际实现中还有很多细节，例如 Word 中还会存储一些 Meta 信息等。
- 时间复杂度
  - 增删查改：O(n)
  - 交：O(\|B1\|+\|B2\|)，其中\|B1\|和\|B2\|表示两个 bitmaps 压缩后的长度；相比于没有压缩的 Hash set 实现：O(min(\|S1\|, \|S2\|))，其中\|S1\|和\|S2\|表示两个集合的 Cardinalities；
  相比于压缩后的（tree）的 Hash set 实现：O(nlogn)。
  - 并差：O(\|B1\|+\|B2\|)
- 缺点
  - 没有 Random access，如果要检查一个元素是否存在，要展开所有的 RLW。
  - 如果要进行交并差运算，仍然要展开所有 RLW，最坏情况的计算复杂度是 O(n)。

## The SOTA: Roaring bitmaps
- 在上述的 EWAH 算法中，bitmaps 结构被安排成一个数组，它是"连续"的，因此压缩算法可以在数组中任意一个地方发生，压缩效率高但是带来的问题是 RLW 的解压。
- RBM 的思路有些不一样，它将集合的所有数据拆分成 Chunks，每个 Chunk 存$$2^{16}$$个数据，且不管存储方法如何，反正一个 Chunk 最多只能放65536个数。
- 那么，什么样的数据会被放到同一个 Chunk 中？数字的高16位相同的数字会被放到同一个 Chunk。考虑32位整数，高16位用来标记它被存到哪个 Chunk，低16位用来标志它被存在这个 Chunk 的哪个位置。
- 这就类似于创建了一个二级索引：插入一个数据，首先检查高16位所在的 Chunk 是否存在，不存在则创建一个，这样原生地进行了一次压缩，不存在的 Chunk 不需要创建。
接着通过低16位来安排它在 Chunk 中的位置。
- 根据论文的描述，Chunk 是被装在 Countainer 里的，Chunk 是更宏观的概念，Container 更具体。根据 Chunk 里面存储的数据量分为两种 Container：Array container 和 Bitmap container。
Array container 很直白，就是一个 Short 类型的有序数组，并且它最多存4096个整数，落到这个 Container 的数据可以用 Binary search 来快速搜索，降低计算复杂度。选择4096是因为4096个Short为8k，
可以放进CPU cache中。
Bitmap container 在 Container 中包含了超过4096个整数时使用，是一个01的 Bitmap，与普通的 Bitmap 一样。
可以知道，Array container 用来存储比较稀疏的数据，而 Bitmap container 用来存储比较稠密的数据。
- RBM 有良好的压缩效率的主要原因我认为是它使用二级索引的思想直接代替 RLE，原本需要 RLE 才能做到的事情在它原生设计下就能解决。它同时又具有良好计算效率的原因是它对 Container 不进行压缩，
因此可以O(1)直接定位一个元素是否存在，增删查改的时间复杂度也为O(1)。
- 关于交并差：RBM的交并差只需要对同一个 Chunk 内的数据操作，这里就涉及 Array container 和 Bitmap container 两两之间四种情况，论文中有详细的介绍，在此不赘述。
