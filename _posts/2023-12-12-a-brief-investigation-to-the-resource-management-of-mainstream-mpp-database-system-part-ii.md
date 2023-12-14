---
layout: post
title: "A Brief Investigation to the Resource Management of Mainstream MPP Database System Part II"
date: 2023-12-12
categories: blog
permalink: /:categories/:year/:month/:title.html
---

书接上回：[A Brief Investigation to the Resource Management of Mainstream MPP Database System](https://zhchuu.github.io/blog/2023/11/a-brief-investigation-to-the-resource-management-of-mainstream-mpp-database-system.html)

上文简单介绍了主流 MPP 系统的资源管理方式和用法，这篇文章浅聊一下 Doris 和 StarRocks 的资源管理代码实现。

本文宏观地去讲解整个 Resource Management 的过程，尝试展示每个模块扮演的角色，没有大段地复制粘贴代码，而且代码是会更新的，我写这篇文章的时候，Doris 就正在 Refine FE 部分的代码，写到一半 Rebase 后，发现描述和代码对不上了，只能再次修改。

对于提到的代码文件、类和函数我都会标识出来，并且后面也会接上“类对象”、“函数调用”等字样去告诉读者这是什么，读者可以根据需要去源码中寻找。

## 1. Doris

### 1.1 FE 部分

描述 WorkloadGroup 的代码在`WorkloadGroupMgr.java`和`WorkloadGroup.java`中。前者作为 Mgr 管理所有 Workload，选择哪个 Workload 是以 String 形式存在 ConnectContext 类里的，看类名这应该是连接级别的环境变量，可以看出确实不支持根据规则自动选择 Workload。后者是单个 Workload 的描述，保存相关配置信息。

并发限制和排队功能在`QueryQueue.java`中，它作为“黑盒检票员”，不保存任何 Query 相关信息，只有申请（getToken 函数）和释放（returnToken 函数）接口，因为它只负责并发控制和排队，不负责资源隔离或分配。可以看出 Doris 的并发限制放在 FE，这会导致它的排队参数只在单 FE 上生效。

Doris 的 FE 和 BE 之间通过 Thrift RPC 通信，信息同步是以发布-订阅的形式完成，发布者为 TopicPublisher 类，接收者为 TopicListener 类，所有发布者和订阅者都需要继承这两个类。FE 在`WorkloadGroupPublisher.java`将 Workload 信息序列化并发布，BE 订阅接收。阅读代码可以看出这里是全量发布，毕竟实例 Workload 的信息量没多大。

### 1.2 BE 部分

前面说到的 BE 接收 Workload 信息在`workload_group_listener.cpp`中实现，接收者只做一件事，就是将 FE 发布的信息和 BE 缓存的信息合并，将新增的补充，将已存在的更新，将不存在的删除。

在 BE 的视角 WorkloadGroup 称为 TaskGroup，封装在 TaskGroup 类里，由 TaskGroupManager 类进行管理（`task_group_manager.h`）。在 BE 中对于 TaskGroup 的 CPU 和 MEM 的控制是分开的，CPU 使用 cgroup 控制，MEM 通过 MemTracker + GC 的形式控制。

#### 1.2.1 MEM
{% highlight C++ %}
// Code: doris/be/src/runtime/query_context.h
// MemTracker that is shared by all fragment instances running on this host.
std::shared_ptr<MemTrackerLimiter> query_mem_tracker;
{% endhighlight %}

Doris 的内存监控模块在`mem_tracker_limiter.h`，每个 QueryContext 类绑定一个 MemTrackerLimiter，Query 任何的 malloc / free 都会在这里被追踪，看注释的描述是基于 TCMalloc Hook 做的。关于 Doris 的 MemTracker，之前的文章中有更多的介绍：[[MemTracker in Doris](https://zhchuu.github.io/blog/2023/12/memtracker-in-doris.html)]

{% highlight C++ %}
// Code: doris/be/src/runtime/task_group/task_group.h
struct TgTrackerLimiterGroup {
    std::unordered_set<std::shared_ptr<MemTrackerLimiter>> trackers;
    std::mutex group_lock;
}
std::vector<TgTrackerLimiterGroup> _mem_tracker_limiter_pool;
{% endhighlight %}
每个 TaskGroup 将跑在自己这里的 Query 对应的内存 Tracker 指针维护起来（`task_group.h`），这样每个 Query 的内存消耗都能统计在 TaskGroup 中。

Doris 进行 GC 的模块在`mem_info.h`，首先扫描所有的 TaskGroup 检查 Memory 是否超量（其实就是检查 Tracker 当前追踪了多少内存），如果超量则触发 GC。这里 GC 还有两种情况，即是否允许内存超用，即 FE 的 WorkloadGroup 配置里的 enable\_memory_overcommit 参数。

#### 1.2.2 CPU

{% highlight C++ %}
// Code: doris/be/src/agent/cgroup_cpu_ctl.h
/*
NOTE: directory structure
1 sys cgroup root path:
/sys/fs/cgroup

2 sys cgroup cpu controller path:
/sys/fs/cgroup/cpu

3 doris home path:
/sys/fs/cgroup/cpu/{doris_home}/

4 doris query path:
/sys/fs/cgroup/cpu/{doris_home}/query

5 workload group path:
/sys/fs/cgroup/cpu/{doris_home}/query/{workload group id}

6 workload group quota file:
/sys/fs/cgroup/cpu/{doris_home}/query/{workload group id}/cpu.cfs_quota_us

7 workload group tasks file:
/sys/fs/cgroup/cpu/{doris_home}/query/{workload group id}/tasks

8 workload group cpu.shares file:
/sys/fs/cgroup/cpu/{doris_home}/query/{workload group id}/cpu.shares
*/
{% endhighlight %}
Doris 通过 CgroupCpuCtl 类（`cgroup_cpu_ctl.h`）读写 /sys/fs/cgroup，一个 TaskGroup 对应一条路径，设置不同的 CPU Shares，我觉得 Doris 对用户的透出的接口命名方式应该也是从这里来的，参数为 cpu_share 而不是类似于 cpu_core 之类的。这个类对象是在 TaskGroupManager 里初始化的，以指针的形式传递给调度器 Scheduler 里（详细信息可以看 TaskScheduler 类和 SimplifiedScanScheduler 类）。

{% highlight C++ %}
// Code: doris/be/src/util/threadpool.cpp
void ThreadPool::dispatch_thread() {
    // ...
    if (_cgroup_cpu_ctl != nullptr) {
        static_cast<void>(_cgroup_cpu_ctl->add_thread_to_cgroup());
    }
    // ... 
}
{% endhighlight %}
ThreadPool 创建线程时通过 CgroupCpuCtl 将对应的线程 ID 写到 cgroup 目录下，即完成 CPU 的控制。


## 2. StarRocks (SR)

### 2.1 FE 部分

整体和 Doris 大同小异，只是名字变成了`ResourceGroup.java`和`ResourceGroupMgr.java`。小异的部分首先是默认存在一个 shortQueryResourceGroup 资源组用于短查询；其次是提供 chooseResourceGroup 函数用于根据 ConnectContext 选择最合适的资源组。有些时候多个资源组都能命中匹配规则，因此这里还有按权重排序的逻辑。

SR 的分类器（Classifier）是相比于 Doris 额外多的功能，在`ResourceGroupClassifier.java`中记录一些用户配置的属性，isSatisfied 函数决定 Query 是否能匹配上当前的资源组。

### 2.2 BE 部分

在 BE 的视角 ResourceGroup 称为 WorkGroup，封装在 WorkGroup 类里，由 WorkGroupManager 类进行管理（`work_group.h`）。

{% highlight C++ %}
// Code: starrocks/be/src/exec/workgroup/work_group.cpp
WorkGroupPtr WorkGroupManager::add_workgroup(const WorkGroupPtr& wg) {
    // ...
    create_workgroup_unlocked(wg, write_lock);
    if (_workgroup_versions.count(wg->id()) && _workgroup_versions[wg->id()] == wg->version()) {
        return _workgroups[unique_id];
    } else {
        return get_default_workgroup_unlocked();
    }
}
{% endhighlight %}
关于 FE / BE 的 WorkGroup 信息同步并不是订阅发布的形式，而是通过 Query 携带给 BE 的，BE 发现 WorkGroup 不存在或者有更新（Version 变化），会实时修改当前的记录。

函数 acquire_running_query_token 是向 WorkGroup 申请允许 Query 的地方，SR 在这里进行并发控制，要么返回可运行，要么返回超过并发限制。SR 调用 acquire_running_query_token 是 Instance 级别的，因此并发限制是全局粒度的。

在`fragment_executor.cpp`中有两个地方与 WorkGroup 有交互，首先是 _prepare_workgroup 函数会将 Query 携带的 WorkGroup 信息同步给 WorkGroupManager；其次是 _prepare_runtime_state 函数会将 WorkGroup 的 Big query 限制放到 QueryContext 中，并且配置好内存追踪器（下文展开描述）。

#### 2.2.1 MEM
{% highlight C++ %}
Status FragmentExecutor::_prepare_runtime_state(/* ... */) {
    // ...
    auto* parent_mem_tracker = wg->mem_tracker();
    // ...
    _query_ctx->init_mem_tracker(query_mem_limit, parent_mem_tracker, /* ... */);
    // ...
{% endhighlight %}
与 Doris 类似，每个 QueryContext 都有一个 MemTracker 类对象共享指针，不同的是，它们并不直接保存在 WorkGroup 中。MemTracker 类是树状结构，因此 WorkGroup 中自己维护一个根结点 Tracker，以子树的形式保存其他 Tracker，这一步骤在初始化每个 QueryContext 的 Tracker 时完成。这样自然的，WorkGroup 可以直接管理到所有 Query 的内存。这块代码在 _prepare_runtime_state 函数中。

MemTracker 类的 check_mem_limit 函数会检查是否超限，在外部的各个地方都能调用者函数，并且通过参数传递告知目前正在执行什么步骤，这样报错信息就比较明显：执行 XXX 步骤时 XXX 内存追踪器超过限制。

#### 2.2.2 CPU

要了解 SR 的 CPU 隔离，首先得了解它的调度方式，在 SR 的技术文章中讲述很详细，在这里只简单讲述。SR 是 Pipeline 的调度模型，Query 被拆分为多个执行单元后进入调度队列等待调度，SR 一般是 NUMA 绑核的，因此会有核数个执行线程，调度队列中的执行单元以一定的规则（多级反馈队列）被分配到线程中，执行后及时让出回到队列。

在`pipeline_driver_queue.h`中，对于 WorkGroup 的 CPU 分配，SR 实现了两级队列，一级队列是 WorkGroup 的优先级（即 CPU 配额），二级队列是原来的多级反馈队列。所以我理解 SR 的调度算法和 Linux 的进程完全公平调度算法类似，类似按时间片轮询调度，每个执行单元都能独享线程时间片，只是频次不同。在 WorkGroupDriverQueue 类里实现了两级队列，每个 WorkGroup 里存储 vruntime 的变量来记录运行时间，单位是纳秒，用 vruntime / cpu\_core_limit 得到的分数标识这个 WorkGroup 占用 CPU 的比例，越小表示占用越少，调度引擎每次选择值最小的 WorkGroup 来调度。

考虑到 CPU 隔离是软隔离，可能存在一些 WorkGroup 没有执行单元需要调度，将 CPU 时间让出给其他 WorkGroup，导致它的 vruntime 非常小；又或者新加一个 WorkGroup 后它的 vruntime 是 0，一旦有 Query 上来它们会长时间占用 CPU 导致其他任务 Starving。为了应对这种情况，调度器会根据当前所有 WorkGroup 里最小的 vruntime 作为参考来调整，找一个合适的值作为真实 vruntime 赋值给这类 WorkGroup，具体实现可以看 _enqueue_workgroup 函数。

#### 2.2.3 Big query

{% highlight C++ %}
// Code: starrocks/be/src/exec/pipeline/pipeline_driver_executor.cpp
// ...
while (true) {
    // ...
    // Check big query
    if (!driver->is_query_never_expired() && status.ok() && driver->workgroup()) {
        status = driver->workgroup()->check_big_query(*query_ctx);
    }
    // ...
}
{% endhighlight %}

在`pipeline_driver_executor.cpp`中有个 while 循环不断检查 Query 是否达到 Big query 限制，如果达到则直接 Cancel 掉。
