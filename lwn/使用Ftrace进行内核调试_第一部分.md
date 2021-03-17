# 使用 Ftace 进行内核调试 —— 第一部分

> [原文链接](https://lwn.net/Articles/365835/)

Ftrace 是直接内置在 Linux kernel 中的跟踪程序，很多 Linux 发行版在他们最新发布的版本中已经启用了 frace 的各种配置。ftrace 带给 Linux 的其中一个好处是能够查看当前 Linux 内核正在发生什么，这使得查找问题区域和跟踪奇怪的 bug 更加容易。

Ftrace 能够显示导致奔溃的事件，这使得有更好的机会找到导致奔溃的确切原因，并帮助开发人员创建正确的解决措施。本文是分为两个部分的系列文章，将介绍使用 Ftrace 来调试内核的各种方法。本文是第一部分，分为以下几块内容：

1. 简要讨论如何设置 Ftrace
2. 使用函数跟踪器
3. 从内核内部写入 Ftrace 缓冲区
4. 在检测到问题时停止追踪的各种方法

Frace 源于两个工具，其中一个是 Ingo Molnar 在 -rt 树中使用的 `“延迟跟踪程序(latency tracer)”`。另一个是我自己的 `logdev` 实用工具，它主要用于 Linux 内核调试。这篇文章将主要描述来自 logdev 的特性，但也将介绍源自 latency tracer 的函数跟踪器。

## 设置 Ftrace

目前 Ftrace 的接口 API 在 Debugfs 文件系统中，它通常挂载在 `/sys/kernel/debug`。为了更容易访问，我通常创建一个 `/debug` 的目录用来挂载该文件系统，你可以随意选择你自己的挂载位置。

在配置完 Ftrace 之后，它将在 debugfs 中创建一个 tracing 的目录。本文将引用这个目录，就好像用户首先将目录更改为 debugfs 中的 tracing 目录，以避免 debugfs 被挂载在何处而产生的混淆

```shell
[~]# cd /sys/kernel/debug/tracing
[tracing]#
```

本文的重点是使用 Ftrace 作为调试工具，Ftrace 的一些设置用于其他目的，如查找延迟或分析系统。为了进行调试，应该启用的配置参数包括：

```makefile
CONFIG_FUNCTION_TRACER
CONFIG_FUNCTION_GRAPH_TRACER
CONFIG_STACK_TRACER
CONFIG_DYNAMIC_FTRACE
```

## 函数跟踪 —— 没有修改的必要

Ftrace 最强大的跟踪器之一是函数跟踪器，它使用 gcc 的 -pg 选项来让内核中的所有函数都调用一个特殊函数 `mcount()`。该函数必须在汇编中实现，因为调用不遵循正常的 C ABI（ABI 是指应用程序二进制接口）。

当配置了 `CONFIG_DYNAMIC_FTRACE` 参数时，在引导时会将调用（指的是 `mcount()`）转换成 NOP，以使系统以 100% 的性能运行。在编译阶段会记录 `mount()` 的调用位置，该列表用于在引导时将这些位置转换为 NOP。由于 NOP 对于跟踪是没有什么用处的，所以保存该列表是为了在启用函数跟踪或者函数图的时候将调用位置转换回跟踪调用。

强烈建议启用 `CONFIG_DYNAMIC_FTRACE`，因为这样可以提高性能。另外，`CONFIG_DYNAMIC_FTRACE` 提供了过滤跟踪函数的功能。注意，尽管 NOP 在 benchmark 测试中几乎没有影响，但 -pg 选项附带的帧指针会带来少量的开销影响。

要找出可用的跟踪器，只需要在 tracing 目录中查看 available_tracers 文件即可：

```shell
[tracing]# cat available_tracers 
function_graph function sched_switch nop
```

要启用函数跟踪器，只需 echo function 到 current_tracer 文件：

```shell
[tracing]# echo function > current_tracer
[tracing]# cat current_tracer
function

[tracing]# cat trace | head -10
# tracer: function
#
#           TASK-PID    CPU#    TIMESTAMP  FUNCTION
#              | |       |          |         |
            bash-16939 [000]  6075.461561: mutex_unlock <-tracing_set_tracer
          <idle>-0     [001]  6075.461561: _spin_unlock_irqrestore <-hrtimer_get_next_event
          <idle>-0     [001]  6075.461562: rcu_needs_cpu <-tick_nohz_stop_sched_tick
            bash-16939 [000]  6075.461563: inotify_inode_queue_event <-vfs_write
          <idle>-0     [001]  6075.461563: mwait_idle <-cpu_idle
            bash-16939 [000]  6075.461563: __fsnotify_parent <-vfs_write
```

标题很好的说明了输出格式。前两项是跟踪的进程名称和 pid；执行跟踪的 cpu 在括号内表示；时间戳是自启动以来的时间，后面跟着函数名称；本例中，跟踪的函数是在其父函数后面紧接着 “<-” 的函数。

这些信息非常强大，可以很好的显示函数调用栈，但可能难以理解。由 Frederic Weisbecker 创建的 `function graph` 跟踪器，可以跟踪函数的入口和出口，从而可以知道函数调用栈的深度。`function graph` 跟踪器可以让人眼更容易看出内核中的函数执行流程：

```shell
[tracing]# echo function_graph > current_tracer 
[tracing]# cat trace | head -20
# tracer: function_graph
#
# CPU  DURATION                  FUNCTION CALLS
# |     |   |                     |   |   |   |
 1)   1.015 us    |        _spin_lock_irqsave();
 1)   0.476 us    |        internal_add_timer();
 1)   0.423 us    |        wake_up_idle_cpu();
 1)   0.461 us    |        _spin_unlock_irqrestore();
 1)   4.770 us    |      }
 1)   5.725 us    |    }
 1)   0.450 us    |    mutex_unlock();
 1) + 24.243 us   |  }
 1)   0.483 us    |  _spin_lock_irq();
 1)   0.517 us    |  _spin_unlock_irq();
 1)               |  prepare_to_wait() {
 1)   0.468 us    |    _spin_lock_irqsave();
 1)   0.502 us    |    _spin_unlock_irqrestore();
 1)   2.411 us    |  }
 1)   0.449 us    |  kthread_should_stop();
 1)               |  schedule() {
```

这给出了一个函数的开始和结束，使用类似于 C 的 “{ }” 来表示。叶函数它不调用其他函数，使用 “;” 结束。DURATION 列显示了相应的函数所花费的时间，`function graph` 记录函数开始和结束的时间，将其差值作为函数的持续时间，DURATION 只会在叶函数和 "}" 一行出现。请注意，这段时间包括所有嵌套函数的开销和 `function graph` 自身的开销。`function graph` 劫持函数的返回地址，为函数的退出插入回调函数。这破坏了 CPU 的分支预测，并导致了比函数跟踪器更大的开销，最接近的真实时间只出现在叶函数上。

孤独的 “+” 是一个注释标记，当持续时间大于 10微秒 的时候会出现 “+”，大于 100微妙 就会出现 “！”。

## 使用 trace_printk

`printk()` 是 debug 之王，但是它有一点问题。如果您正在调试一个大型区域，例如定时器中断、调度程序或者网络，`printk()` 可能会导致系统陷入停顿，甚至可能创建一个活动锁。当添加一些 `printk()` 的时候 bug 消失了也是有可能的，这是因为 `printk` 引入了大量的开销。

Ftrace 引入了一种名为 `trace_printk()` 的新的 `printk` 形式。它可以像 `printk` 那样使用，也可以在任何上下文中使用（中断、NMI 和调度）。`trace_printk` 的好处在于它不会写入控制台，与之相对的，它写入 Ftrace 环形缓冲区，并可以通过 trace 文件读取。

用 `trace_printk` 写入环形缓冲区，只需要大约十分之一微秒。但是使用 `printk`，尤其是写入串形控制台，每次写入可能需要花费几毫秒。`trace_printk` 的性能优势使您可以记录内核中最敏感的区域，而影响很小。

例如，你可以像这样往内核或者模块中添加内容：

```C
trace_printk("read foo %d out of bar %p\n", bar->foo, bar);
```

然后通过查看 trace 文件，你可以看到输出：

```shell
[tracing]# cat trace
# tracer: nop
#
#           TASK-PID    CPU#    TIMESTAMP  FUNCTION
#              | |       |          |         |
           <...>-10690 [003] 17279.332920: : read foo 10 out of bar ffff880013a5bef8
```

上面的例子是通过添加一个实际具有 foo 和 bar 构造的模块来完成的。

`trace_printk` 出现在任何跟踪器中，甚至在 function 和 function graph 跟踪器中。

```shell
[tracing]# echo function_graph > current_tracer
[tracing]# insmod ~/modules/foo.ko
[tracing]# cat trace
# tracer: function_graph
#
# CPU  DURATION                  FUNCTION CALLS
# |     |   |                     |   |   |   |
 3) + 16.283 us   |      }
 3) + 17.364 us   |    }
 3)               |    do_one_initcall() {
 3)               |      /* read foo 10 out of bar ffff88001191bef8 */
 3)   4.221 us    |    }
 3)               |    __wake_up() {
 3)   0.633 us    |      _spin_lock_irqsave();
 3)   0.538 us    |      __wake_up_common();
 3)   0.563 us    |      _spin_unlock_irqrestore();
```

是的，`trace_printk()` 输出看起来像 function graph 跟踪器中的注释。

## 开始和结束跟踪

显然，有些时候你只想跟踪特定的代码路径，也许你只想在运行特定测试的时跟踪正在发生的事情，tracing_on 文件用于禁止环形缓冲区记录数据：

```shell
[tracing]# echo 0 > tracing_on
```

这将禁止 Ftrace 环形缓冲区记录数据。所有其他事情仍然会发生在跟踪器上，他们依旧会导致大部分的开销。它们确实注意到环形缓冲区不会记录数据，它们也不会写数据，但是跟踪器发出的调用仍然会执行。

要重新启用环形缓冲区，只需往文件中写一个 "1"：

```shell
[tracing]# echo 1 > tracing_on
```

注意数字与 ">" 之间的空格十分重要，否则你可能正在标准输入或输出写入该文件。

```shell
[tracing]# echo 0> tracing_on   /* this will not work! */
```

常见的运行方式可能是：

```shell
[tracing]# echo 0 > tracing_on
[tracing]# echo function_graph > current_tracer
[tracing]# echo 1 > tracing_on; run_test; echo 0 > tracing_on
```

第一行禁止环形缓冲区记录任何数据；下一行启动 function_graph 跟踪器，function_graph 的开销依然存在但是不会往环形缓冲区记录数据；最后一行启动环形缓冲区，运行测试程序，然后关闭环形缓冲区。这将缩小 function_graph 记录的数据范围，使其主要包括 run_test 产生的数据。

## 下一步是什么

下一篇文章将继续讨论使用 Ftrace 调试内核。上面禁用跟踪的方法可能不够快，程序 run_test 的结束和 `echo 0 > tracing_on` 之间的延迟可能会导致环形缓冲区溢出并丢失相关数据。我将讨论其他更有效的停止跟踪的方法，如何调试奔溃，以及查看内核哪些函数占用堆栈。想要了解更多的信息，最好的方法是启用 Ftrace 并使用它，通过 function_graph 跟踪器，你可以了解很多关于内核如何工作的信息。