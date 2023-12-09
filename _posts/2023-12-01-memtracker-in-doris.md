---
layout: post
title: "MemTracker in Doris"
date: 2023-12-01
categories: blog
permalink: /:categories/:year/:month/:title.html
---

Doris 的内存管理基础模块是 MemTracker 类，它非常精炼和易读，四个文件（mem\_tracker.h / cpp & mem\_tracker_limiter.h / cpp）实现了所有内容，是个不错的学习范本。

MemTracker 顾名思义就是内存追踪，Doris 的内存申请用的是 tcmalloc 库，因此 MemTracker 是利用 TCMalloc Hook 来实现自动追踪的。

内存消耗和释放无非是值的加减，在 mem_tracker.h 中首先定义了 MemCounter 类作为内存计数器：

{% highlight C++ %}
// Code: mem_tracker.h
// A counter that keeps track of the current and peak value seen.
// Relaxed ordering, not accurate in real time.
class MemCounter {
public:
    MemCounter() : _current_value(0), _peak_value(0) {}
    // ...
private:
    std::atomic<int64_t> _current_value;
    std::atomic<int64_t> _peak_value;
};
{% endhighlight %}

为了性能，它的实现是无锁的，内存也使用相对宽松的 std::memory\_order\_relaxed 策略，这意味着编译器不需要为它添加额外的内存屏障维持操作顺序，能使用更激进的优化，对性能更友好。牺牲的是瞬时统计值的准确性，但这点微小的不准确对内存统计来说无伤大雅。

{% highlight C++ %}
enum class Type {
    GLOBAL = 0,        // Life cycle is the same as the process, e.g. Cache and default Orphan
    QUERY = 1,         // Count the memory consumption of all Query tasks.
    LOAD = 2,          // Count the memory consumption of all Load tasks.
    COMPACTION = 3,    // Count the memory consumption of all Base and Cumulative tasks.
    SCHEMA_CHANGE = 4, // Count the memory consumption of all SchemaChange tasks.
    CLONE = 5,         // Count the memory consumption of all EngineCloneTask. Note: Memory that does not contain make/release snapshots.
    EXPERIMENTAL = 6   // Experimental memory statistics, usually inaccurate, used for debugging, and expect to add other types in the future.
    };

{% endhighlight %}
MemTrackerLimiter 类继承了 MemTracker 类，个人理解是为了：1. 一定程度赋予业务逻辑；2. 顾名思义，增加内存使用限制的逻辑。对于第 1 点，MemTrackerLimiter 类里定义了一些业务相关的 Type，例如 QUERY、COMPACTION 和 SCHEMA_CHANGE，用来表示这个 Tracker 用于追踪哪个阶段的内存。对于第 2 点，提供 check\_limit 函数用于检查内存是否超过限制，提供 free\_xxx\_query 函数用于清理超限制的 Query，类中的 \_label 变量包含 query\_id，用它能解析出 Query 并 Cancel。

{% highlight C++ %}
// Code: mem_tracker.h
struct TrackerGroup {
    std::list<MemTracker*> trackers;
    std::mutex group_lock;
};
static std::vector<TrackerGroup> mem_tracker_pool;
{% endhighlight %}
MemTracker 是多层树状结构，根据官方技术文档的描述，按层次自上而下分为 Process / Query / Instance / ExecNode，我理解分别表示：进程相关 / Query 相关 / Fragment instance 相关 / 算子相关。当然了这些都是业务逻辑，MemTracker 的实现方式是用一个名为 TrackerGroup 的结构体表示一层，TrackerGroup 结构体内用 std::list 存储这一层所有 Tracker，变量是 static 全局的。

MemTracker 具体使用的地方在 ThreadContext 类里，ThreadMemTrackerMgr 类对象管理线程所有的 Tracker，当线程需要申请内存时 Mgr 调用所有 Tracker 的 consume 函数记录上，让检查内存的逻辑放在外面，不和 Thread 牵扯在一起，Thread 只管申请和释放内存。

尽管 MemCounter 是并发无锁的，频繁地调用 Tracker 的开销还是不小，因此 Mgr 做了一个攒批 Flush 的逻辑，防止 Tracker 成为瓶颈。这部分代码有一些地方值得学习，是我在工程实践中没意识到的点。Mgr 是用一个 \_untracked\_mem 变量来攒批的，push\_consumer\_tracker 函数用于添加一个新的 Tracker 追踪内存：
{% highlight C++ %}
// Code: thread_mem_tracker_mgr.h
inline bool ThreadMemTrackerMgr::push_consumer_tracker(MemTracker* tracker) {
    // ...
    if (/* exists */) {
        return false;
    }
    _consumer_tracker_stack.push_back(tracker);
    tracker->release(_untracked_mem);  // Why
    return true;
}
{% endhighlight %}
这段逻辑很简单，首先检查 Tracker 是否存在，如果不存在则添加。但是最后有一个 tracker->release 的操作，第一眼没看懂原因，仔细想想是因为 \_untracker\_mem 是攒批 Flush 的，这里如果不提前 Release 会使得下次 Flush 的时候把这些内存放到新加的 Tracker 里，会造成多统计一部分，因为它们是新 Tracker 添加之前申请的。添加 Tracker 如此，那么移除 Tracker 也是这样，移除之前要把 \_untracked\_mem 算进去，不然就会少统计。

在 Flush 时要考虑防止递归的情况发生：
{% highlight C++ %}
// Code: thread_mem_tracker_mgr.h
inline void ThreadMemTrackerMgr::flush_untracked_mem() {
    // ...
    _stop_consume = true;
    old_untracked_mem = _untracked_mem;
    // ...
    // Do the consume...
    // ...
    _untracked_mem -= old_untracked_mem;
    _stop_consume = false;
}
{% endhighlight %}
这里用 \_stop\_consume 变量来保护 Flush 只进行一次，如果变量为 true，则即使 \_untracked\_mem 超过限制也不会 Flush，这么做是因为 Flush 过程本身也可能有内存申请，如果 Flush 的阈值非常小，那么 Flush 动作本身就会触发 Flush，进入无限循环中，因此需要一个 Flag 来保护。


### Reference:
[Refactor memory tracker on BE](https://cwiki.apache.org/confluence/display/DORIS/DSIP-002%3A+Refactor+memory+tracker+on+BE)  \\
[Doirs新MemTracker](https://shimo.im/docs/DT6JXDRkdTvdyV3G/read)
