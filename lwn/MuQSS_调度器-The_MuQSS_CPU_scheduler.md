# MuQSS 调度器

> 原文链接 [MuQSS 调度器](https://lwn.net/Articles/720227/)，
> 配合以下两篇文章食用更佳：
> [返璞归真的Linux BFS调度器](https://blog.csdn.net/dog250/article/details/7459533)，
> [Linux 调度器 BFS 简介](https://www.ibm.com/developerworks/cn/linux/l-cn-bfs/index.html)

因为调度算法对 Linux 桌面操作系统的整体响应性能有相当大的影响，所以桌面系统用户对调度器总是非常感兴趣。Con Kolivas 维护了一系列调度器补丁集，多年以来，他根据自己的使用情况对这些补丁集做了很多调整，主要为了减少调度延迟以优化桌面体验。2016 年 10 月初，Kolivas 更新了他广受欢迎的桌面调度器 BFS 的设计，并将其重命名为 MuQSS，新的设计是为了解决 BFS 在 CPU 数量不断增加时遇到的可伸缩性问题（scalability concerns）。

## BFS

为了理解 MuQSS 调度器中的新东西，我们先来看一下 BFS 的设计，它是 MuQSS 的基础。BFS 于 2009 年首次发布，它是一个非常简单的调度器，针对主线调度器在桌面环境下使用时遇到的问题而开发。主线调度器必须在合理处理 CPU 受限（[CPU-bound](https://www.quora.com/What-is-the-meaning-of-the-CPU-bound)）任务时，保证桌面的流畅敏捷，这种平衡是很难实现的（注：同时满足大吞吐量和低延迟很难实现）。Kolivas 提出，没有一个调度器能适应所有情况，所以他专门为桌面环境设计了自己的调度器。

BFS 的任务队列是全局唯一的。所有进程的优先级只根据其 nice 值进行设置，并且不会由于睡眠行为造成优先级变动。Kolivas 认为任何进程交互性估计算法都不可靠，他们在寻找交互性进程时总有误判。选择单运行队列设计是因为 Kolivas 想要调度程序只做全局决策而不考虑进程创建的原始 CPU，同时避免遍历多个不同的队列来查找要运行的下一个进程。

BFS 使用最早合格虚拟截止时间优先（earliest eligible virtual deadline first，EEVDF）算法来决定何时运行哪些任务。每个进入运行队列的任务都有一个时间片（time slice）和一个截止时间（deadline）。时间片（rr_interval）确定每个任务分配的 CPU 运行时间。截止时间等于当前时间加上时间片与优先级的乘积，其中优先级由 nice 值决定。具有较低 nice 值（因此具有较高优先级）的进程的截止时间比较低优先级的截止时间要早。因此，在较低优先级任务的达到截止时间之前，高优先级任务可能会运行很多次。当任务的时间片用完时，将计算新的截止时间重新入队。如果任务从睡眠状态唤醒或被更高优先级的任务抢占，放回运行队列中，并不改变其截止时间，在下一轮调度中将再次处理该任务。

时间片默认设置为 6ms，因为人类感知抖动的阈值刚好不到 7ms。Kolivas 设想的一个常见的情况是，每个 CPU 有 0 到 2 个正在运行的任务，则每个任务调度延迟不超过 6ms 就能开始运行。

BFS 没有显式的 CPU 间负载均衡算法，只在计划唤醒进程时进行合理的 CPU 选择。全局运行队列将为下一个准备运行的进程选择可用的空闲 CPU。按以下优先级顺序选择：

1. 进程最后运行的 CPU
2. 相同 CPU 芯片，同一物理核心上的另一个逻辑核心或兄弟物理核心（hyperthreaded or core siblings），因为他们之间共享缓存
3. 不同 CPU 芯片或不同 NUMA 节点的 CPU，因为他们的高速缓存很远（cache distant）

BFS的其他简化设计包括：取消了大多数可配置项，对组调度（control groups）支持很少。Kolivas 认为这些是桌面环境用户不感兴趣的功能，过多的可配置项只会造成混乱。

## MuQSS

Kolivas 指出，由于 BFS 的唯一运行队列对所有 CPU 持有全局锁，当 CPU 数量增加到 16 以上时，锁的竞争十分激励。而且每次调度器选择下一个运行任务时，需要扫描整个运行队列，寻找截止时间最早的进程。这种遍历链表的行为会导致高速缓存崩溃（cache-thrashing）。

可以把 MuQSS 看做是每个 CPU 一个运行队列的 BFS。每个运行队列用跳过列表（[skip lists](https://en.wikipedia.org/wiki/Skip_list)）代替链表实现。跳过列表由 William Pugh 于 1990 年首次提出，它是一种概率数据结构(probabilistic data structures)，它具有平衡二叉树（例如[CFS](https://lwn.net/Articles/230574/)中使用的红黑树）的一些性能优势，但在确定性效率（deterministically efficient）较低时其实现的计算成本更低。Kolivas 为其调度程序自定义实现了跳过列表，其中的“levels”是静态的八入口数组（eight-entry arrays）。这些数组的大小恰好是一个高速缓存行宽。

MuQSS 计算截止时间时，使用“niffies”来跟踪当前时间。Niffies 是精度为纳秒的单调递增计数器，其值是从高分辨率 TSC 计时器获得的。Niffies 在 2010 年某时引入 BFS，用来代替原先计算截止时间所用的 Jiffies。MuQSS 的虚拟截止时间算法与 BFS 基本相同：

virtual_deadline = niffies + (prio_ratio * rr_interval)

prio_ratio 是由 nice 值经标准化计算得出的比率，rr_interval 是由 nice 值对应的时间片长度。插入任务时，将根据其虚拟截止时间的值按某种顺序插入运行队列。由于队列是排序的，调度器可在 O(1) 时间找到下一个符合条件的任务来运行，因为该任务位于队头。插入运行队列需要 O(log n) 时间。

选择要运行的任务时，调度程序将对每个 CPU 的运行队列进行无锁检查，以找到符合条件的任务以运行，并选择截止时间最近的任务。从相关的运行队列中弹出选定的任务时，调度程序将使用非阻塞的“trylock”方法，即如果未能获取队列锁，将继续选择另一个队列的下的最近截止时间任务。尽管这意味着调度器有时无法选择全局上截止时间最近的任务去运行，但是它确实缓解了不同 CPU 队列之间的锁争用问题。

像 BFS一样，MuQSS 知道 CPU 逻辑核心的拓扑结构，包括超线程核心层级，物理核心层级，NUMA 层级。将在超线程同级CPU之前选择一个空闲内核（如果可用），以避免由于CPU的处理能力在同一内核上的任务之间共享而导致速度降低。当两个任务在一个超线程核心上执行时，为了让优先级较高的任务获得更多的 CPU 时间，优先级较低的任务将受到时间限制。还包括一个称为“SMT Nice”的功能，该功能在超线程核心层根据优先级按比例分配 CPU 时间。

MuQSS 和 BFS 一样引入了两个新的调度策略：SCHED_ISO 和 SCHED_IDLEPRIO。 SCHED_ISO（等时调度，isochronous scheduling），允许非特权进程采用准实时的调度行为（注：用于交互进程）。设置为 SCHED_ISO 的任务会抢占 SCHED_NORMAL 任务，只要这个任务的步长为 5 秒的移动平均 CPU 使用率不超过 70％，这个阈值是可调的。另一方面，同时运行的多个 SCHED_ISO 任务之间采用轮转执行（round-robin）。如果一个 SCHED_ISO 任务的平均 CPU 使用率超过 70％ 阈值，它将暂时降级使用 SCHED_NORMAL 调度，直到经过足够的时间使其平均 CPU 使用率再次降至 70％ 以下。由于系统上每个 CPU 都有 70% 阈值，所以 SMP 架构的多核机器上完全可以使用任一可用的 CPU 来执行 SCHED_ISO 任务。例如，在双核机器上，执行一个 SCHED_ISO 任务最多只能消耗 CPU 总能力的 50%（注：两个 CPU 被占用一个）。

SCHED_IDLEPRIO 与 SCHED_ISO 相反，它强制任务具有超低优先级。尽管它与主线调度程序中的 SCHED_IDLE 相似，但还是有细微的差别：即使同时有其他优先级更高的任务在运行，主线调度程序仍将最终运行 SCHED_IDLE 任务，但是 SCHED_IDLEPRIO 任务仅在绝对没有其他情况下运行 需要CPU的注意。 这对于清除空闲CPU（例如SETI @ Home）的批处理任务很有用。 为了避免死锁，如果 SCHED_IDLEPRIO 任务正在保存共享资源（例如互斥量或信号量），则即使队列中还有其他更高优先级的进程，该任务也会被临时提升回SCHED_NORMAL，以使其能够运行。

下面三个可调参数可以控制调度器的行为：

1. `rr_interval`，时间片长度，默认为 6ms。
2. `interactive`，截止时间行为开关。如果禁用此选项，则选择下一个要运行的任务时，只遍历本地 CPU 的运行队列，而不是遍历所有 CPU。
3. `iso_cpu`，对于 SCHED_ISO 调度策略的进程，其步长为 5 秒的移动平均 CPU 使用率阈值

## 与主线内核的比较

Kolivas 尚未也不会尝试将他的调度程序合并到主线内核中，因为他的调度程序是为他自己的桌面使用场景量身定制的。然而，他的补丁集受到了一些 Linux 桌面用户的追捧，他们报告说使用它会带来“更流畅”的桌面体验。使用传统的吞吐量基准测试比较 BFS 与 CFS，并不能说明哪一个更好。但有报告称，MuQSS 调度程序的 [Interbench](http://ck-hack.blogspot.com/2016/10/interbench-benchmarks-for-muqss-116.html) 基准评分比 CFS 好。总之，我们很难去量化“平滑度”和“响应度”这种指标，并将他们转换为自动基准测试，因此，想要去评估 MuQSS 性能的最佳方法就是自己[尝试一下](http://ck.kolivas.org/patches/muqss/)。
