# 基于页面的直接I/O

> [原文链接](https://lwn.net/Articles/348719/)

内核术语中的“地址空间”是地址范围及其在底层文件系统或设备中的表示形式之间的映射。每个打开的文件都有一个地址空间，任何给定的地址空间可能会也可能不会绑定到进程的虚拟地址空间中的虚拟内存区域。在一个典型的进程中，将存在许多地址空间，用于正在运行的可执行文件的映射、该进程的打开文件和匿名内存映射（匿名内存映射使用 swap 作为其后备存储）。

进程有许多方式操作它的地址空间，其中有一种就是直接 I/O 。Jens Axboe 的新补丁系列希望对直接 I/O 路径进行合理化处理，使其在进程中更为灵活。

直接 I/O 的想法是，数据块直接在存储设备和用户内存之间移动而不经过页面缓存（page cache）。开发者使用直接内存的原因有以下两个（之一或同时）：**(1)** 他们认为比起内核，他们更能管理文件内容的缓存。**(2)** 它们希望避免在不久的将来不太可能用到的数据导致页面缓存溢出。这是一个很少用到的功能，它通常和内核另一个晦涩的功能结合使用：异步 I/O。到目前为止，此功能最大使用者是大型关系型数据库系统，因此 Oracle 当前雇佣的开发人员在这一领域工作并不奇怪。

当内核需要对地址空间操作时，它通常会在相关的 `address_space_operations` 数据结构中查找合适的函数。例如，普通文件 I/O 的处理方式为：

```C
int (*writepage)(struct page *page, struct writeback_control *wbc);
int (*readpage)(struct file *filp, struct page *page);
```

与大量低级的、面向内存的内核操作一样，这些函数在 page 上操作。在此级别管理内存时，不需要担心这是用户空间还是内核空间，也不需要担心这是不是在高端内存区域。它们全部都是内存。处理直接 I/O 的函数看起来有一些不同：

```C
ssize_t (*direct_IO)(int rw, struct kiocb *iocb, const struct iovec *iov, loff_t offset, unsigned long nr_segs);
```

`kiocb` 结构体的使用显示了一个假设，直接 I/O 将通过异步 I/O 的路径提交。但是，除此之外，指向要输出的缓冲区的 `iovec` 结构直接来自于用户空间，包含用户空间地址。这又意味着 `direct_IO()` 函数本身必须处理访问用户空间缓冲区的过程。该任务通常在 `VFS` 层通用代码中处理，但是还有另一个问题： `direct_IO()` 函数无法在内核空间被调用。

内核本身通常不需要使用直接 I/O，但是有一个例外：回送驱动程序（loopback driver）。该驱动程序允许将普通文件作为块设备挂载，它对于访问在磁盘文件中存储的文件系统映像最有用。但是访问通过 loopback 挂载的文件很可能在页面缓存中表示两次：loopback 挂载的每一侧。结果是浪费内存，可以更好的利用它们。

总之，最好更改 `direct_IO()` 接口以避免这种内存浪费，并使它与其他地址空间操作更加一致。这就是 [Jens 的 patch](https://lwn.net/Articles/347371/) 做的。在他的 patch 中，接口变成了：

```C
struct dio_args {
	int rw;
	struct page **pages;
	unsigned int first_page_off;
	unsigned long nr_segs;
	unsigned long length;
	loff_t offset;

	/*
	 * Original user pointer, we'll get rid of this
	 */
	unsigned long user_addr;
    };

ssize_t (*direct_IO)(struct kiocb *iocb, struct dio_args *args);
```

在新的 API 中，许多相关参数已分组为 `dio_args` 结构。可以通过 `pages 数组` 找到要传输的内存。现在，更高级别的 VFS 直接 I/O 代码可以处理映射用户空间缓冲区和创建页面数组的任务。

在大多数情况下，对代码的影响很小，这主要是移动用户空间地址到页面结构的转换的位置。当前的代码确实存在潜在的问题，因为它一次仅能处理一个 I/O 段，可能会为某些类型的应用程序造成性能问题。但是，这种操作并没有真正连接到系统中，并且大概可以固定在某一个位置。

唯一的[反对意见](https://lwn.net/Articles/348733/)来自 Andrew Morton ，他不喜欢 Jens 的通过 page 数组进行工作的实现方式。

该数组的索引（称为 `head_page`）内置于 `struct dio` 中，并从实际在页面中工作的代码中隐藏，这会导致潜在的混乱，尤其是在操作中途中断的情况下。Andrew 称其为“等待发生的灾难”，并建议在处理页面数组的位置明确建立索引。

不过，这是一个细节——尽管可能很重要。 核心目标和实施似乎收到了很好的评价。这段代码似乎不太可能在 2.6.32 合并准备就绪，但是我们可能会看到它针对后续开发周期中的主线。