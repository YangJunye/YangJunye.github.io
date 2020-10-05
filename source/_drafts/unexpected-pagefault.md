---
title: 记一次迷之Page Fault定位
tags: [NUMA, Linux]
categories: 技术随笔
date: 2020-09-26 14:18:51
---

# 太长不看版

NUMA Balancing会定期扫描内存地址，在不同的NUMA Node之间搬运，造成Page Fault。

# 起因

场景是我们的项目有一个巨大的全局`ConcurrentHashMap<Key, Value>`，用拉链法解决冲突。在做纯`Insert`的性能测试的时候发现，[火焰图](https://github.com/brendangregg/FlameGraph)显示它的`Find`方法中存在大量Page Fault。一般来说，造成Page Fault的原因有：
<!-- more -->


仔细阅读代码逻辑后觉得并不是以上任何一个，于是换用`-O0`编译，希望看到更完整的调用栈。重新生成火焰图后发现是`Key`的比较函数`operator==`触发的。这就很神奇了，既然触发了比较函数，意味着这个`key-value`一定是之前插入的，也就是这块内存已经被使用过了。而物理内存非常充足，并且`swap`也关闭了，不存在内存页被换出去的情况。
```
cat /proc/sys/vm/swappiness
```
> swappiness=0的时候表示最大限度使用物理内存，然后才是swap空间，swappiness=100的时候表示积极的使用swap分区。众所周知，内存的速度比磁盘快得多，swap空间的使用会加大系统IO，同时造成大量页的换进换出，严重影响系统的性能。

# 进一步排查

接着我又hack了`operator==`，让它直接返回`false`（`Insert`每次的`Key`都不相同，不影响正确性）。这回火焰图里的`operator==`以及它上面的`page_fault`确实消失了，但`Find`本身出现了大量`page_fault`。因此很自然地想到是遍历溢出链导致的问题。然后又hack了遍历的长度，发现`page_fault`的数量和它是正相关的。

# NUMA

# NUMA Balancing

[文章](https://doc.opensuse.org/documentation/leap/tuning/html/book-sle-tuning/cha-tuning-numactl.html)
> Automatic NUMA balancing happens in three basic steps:
> 1. A task scanner periodically scans a portion of a task's address space and marks the memory to force a page fault when the data is next accessed.
> 2. The next access to the data will result in a NUMA Hinting Fault. Based on this fault, the data can be migrated to a memory node associated with the task accessing the memory.
> 3. To keep a task, the CPU it is using and the memory it is accessing together, the scheduler groups tasks that share data.
>
> The unmapping of data and page fault handling incurs overhead. However, commonly the overhead will be offset by threads accessing data associated with the CPU.

然后看了一下服务器的参数
```bash
cat /proc/sys/kernel/numa_balancing
```
果然是`1`，坑……
到这里基本已经确定是NUMA Balancing的锅了。为了进一步盖棺定论，又做了以下的测试：
```bash
# 增大 NUMA Balancing 扫描间隔
echo 3000 > /proc/sys/kernel/numa_balancing_scan_period_min_ms
# 结果: Find中的page_fault减少

# 增大 NUMA Balancing 扫描间隔
echo 5000 > /proc/sys/kernel/numa_balancing_scan_period_min_ms
# 结果: Find中的page_fault减少

# 关闭 NUMA Balancing
echo 0 > /proc/sys/kernel/numa_balancing
# 结果: Find中的page_fault完全消失
```

# 总结
