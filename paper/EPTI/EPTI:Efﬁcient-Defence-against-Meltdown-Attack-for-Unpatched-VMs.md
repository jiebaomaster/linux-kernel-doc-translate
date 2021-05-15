**摘要**

​        Meltdown 漏洞利用了 x86、ARM 和 PowerPC 等常见处理器中固有的无序执行，打破了用户和内核空间之间的基本隔离边界。现代操作系统因为 Meltdown 漏洞更新了一个不小的补丁，用于将用户空间和内核空间页表分离，即 KPTI（kernel page table isolation）。虽然这个补丁阻止了恶意用户进程的内核内存泄漏，但它要用户修补其内核（这通常需要重启），并且目前只在最新版本的 OS 内核上可以使用。此外，由于在内核态和用户态切换的时候要进行页表切换，它还引入了不小的性能开销。

​        在本文中，我们提出了 EPTI，一种可替代的方法，来防范 cloud 中未打补丁的虚拟机的 Meltdown 攻击，并且它的性能比 KPTI 要好。具体来说，我们不是使用两个页表（内核和用户），而是使用两个 EPT（扩展页表）来隔离用户空间和内核空间，并取消用户 EPT 中的所有内核空间的映射，以实现 KPTI 的相同工作。EPT 的切换是通过一个硬件支持特性来完成的（称为 EPT 切换），不涉及虚拟机管理程序的干扰。同时，每个 EPT 都有自己的 TLB，因此切换 EPT 不会刷新 TLB，从而进一步减小了开销。我们已经实现了我们的设计，在 Intel Kaby Lake CPU 和不同版本的 Linux 内核上进行了评估。结果表明，EPTI 只引入了高达 13% 的开销，比 KPTI 少了大约 45%。

# 1 Introduction

​        最近发现的 Meltdown 和 Sprctre 漏洞允许未经授权的进程读取特权内核或其他进程的数据，这给云平台带来了严重的安全威胁。目前，英特尔已经发布了微代码补丁来修复 Spectre 漏洞。但是，为了修复更严重，更容易利用的 Meltdown 漏洞，用户需要应用名为 KPTI（内核页面表隔离）的内核补丁，该补丁使用两个页表表示内核和用户程序。隔离内核地址空间和用户进程的地址空间。虽然这个补丁可以有效地防御 Meltdown 攻击，但它带来了三个问题，这使得成千上万未打补丁的机器处于危险之中。

​        首先，每个用户都必须手动应用补丁。在云环境中，尽管云管理员可以给主机操作系统打补丁，但他们不能直接给运行在 vm(虚拟机) 中的客户操作系统打补丁，因为不允许他们这样做。例如，Amazon “建议客户修补他们的实例操作系统，以解决进程到进程或进程到内核的问题”。然而，许多云用户没有能力进行这样的系统维护。

​        其次，补丁可能依赖于特定版本的内核，特别是 Linux。到目前为止，Linux 社区刚刚发布了包含该补丁的 4.15 版本。该补丁可能无法在一些早期版本的内核上工作，比如 4.4。预计该补丁将需要很长时间才能应用于所有版本的 Linux 内核。

​        第三，补丁可能会导致严重的性能下降。KPTI 补丁使内核和用户进程使用不同的页表，这导致在用户模式和内核模式之间切换时会刷新 TLB，从而增加 TLB miss 的比率。之前的评估结果表明，对于某些系统调用密集型工作负载，vm 中的性能损失可能高达30%；我们自己的实验证实了这种性能下降(第 6 节)。

​        在这篇论文中，我们提出了一种替代的方法来防范云中的虚拟机的 Meltdown 攻击。我们的方法，即 EPTI，可以在用户不知情的情况下应用于未打补丁的客户虚拟机，同时可以获得比 KPTI 更好的性能。

- 首先，与 KPTI 中使用两个 gPTs(guest page table)不同，EPTI 使用两个 EPT(扩展页表)，即 EPTk 和 EPTu，相应地运行内核进程和用户进程。guest 内核和用户仍然共享一个 gPT，但在用户模式下，映射内核地址空间的 gPT 条目在 EPTu 中被设置为零，这禁止了内核空间内的任何地址转换，以减少 Meltdown 攻击。
- 其次，我们利用英特尔的一种硬件功能（称为 EPT switching）进行虚拟化，在虚拟机本身内切换两个 EPT，而不会导致任何 VMExit。我们使用二进制工具在 guest 内核的入口和出口插入两个 trampolines 来进行 EPT 切换，这不需要内核的源代码，并且对内核版本的依赖性很小（如果有的话）。
- 第三，通过详细的微观结构分析，我们发现 EPT 切换比 gPT 切换更有效。由于每一个 EPT 都有自己的 TLB，因此在切换 EPT 时不会出现硬件冲刷 TLB 的情况，这是 KPTI 性能下降的主要原因。我们还采用了几种优化措施来最大程度地减少 VMExits 的数量，从而进一步减少开销。
- 第四，通过与实时 VM 迁移相结合，EPTI 可以无缝地部署在云中：主机可以迁移掉所有 guest VM，用 EPTI 修补主机虚拟机监控程序，然后将所有 VM 迁移回来。

​        我们已经实现了 EPTI KVM，并使用未修改的 Ubuntu 发行版作为客户虚拟机进行评估。我们进行了详细的安全分析和评估，以表明我们的 EPTI 可以实现与 KPTI 相同的安全保证。我们还将评估实际 benchmark，以衡量性能开销。结果表明，EPTI 在服务器应用上的平均性能开销约为 6%，比KPTI的平均性能开销 11% 降低了 45%。

​        综上所述，本文做出了以下贡献：

- EPT 级别的内核和用户地址空间隔离，可以防御未修补的 guest VM 的 Meltdown 攻击。
- 实现比当前解决方案KPTI更好的性能的几个优化。
- 我们在真实硬件上进行设计的原型，以进行性能和安全性评估。

# 2 动机和背景

## 2.1 Meltdown 攻击和 KPTI

​        Meltdown 漏洞发布于 2018 年 1 月，已知影响英特尔 x86 CPU、ARM CortexA75 和 PowerPC 处理器的一些版本。通过这种攻击，恶意用户应用程序可以通过两步窃取内核内存的内容。

- 步骤1：访问内核地址 A，并利用其数据作为索引来访问缓存;
- 步骤2：通过缓存隐蔽通道获取数据。

这里的关键问题是，步骤1 是重新排序执行的，最终将被取消，但是如果没有回滚，缓存布局将受到影响。由于内核通常会在其内存空间中映射所有物理内存，因此恶意应用程序可能获得所有内存内容。

​        KPTI (内核页表隔离)基于 KAISER(内核地址隔离，有效地删除侧通道)，该方法是为了防御 Meltdown 攻击而提出的。这个补丁将用户空间和内核空间页表完全分离，如图 1 所示。

<img src="/Users/li/Library/Application Support/typora-user-images/image-20210510193017585.png" alt="image-20210510193017585" style="zoom:50%;" />

内核使用的映射是和以前一样的，而应用程序使用的映射是包含一个用户空间的副本和一小组内核空间映射，仅使用 trampoline 代码进入内核。由于内核的数据不再映射到用户空间中，恶意应用程序无法直接解引用内核的数据地址，从而无法发动Meltdown攻击。KPTI 已经被合并到主流 Linux 内核 4.15 中，该内核于 2018 年 1 月 28 日发布。然而，该补丁在以前的 Linux 内核版本上仍然存在问题。例如，有报道称一些 Ubuntu 用户“刚刚得到内核 linux-image-4.4.0-108generic 的 Meltdown 更新，但这根本不能启动”。考虑到补丁需要由系统管理员手动应用，因此大多数机器可能需要很长时间才能部署补丁。

> trampoline 代码就是一个 内核地址空间的 PGD，它只包含用户空间所需的最小的内核映射；当用户空间通过系统调用或者其他进入内核时，首先通过 trampoline 的 mapping，接下来 trampoline 的处理会把 mapping 转换成正常的 mapping，并跳转到 kernel 的 vector entry，继续处理
>
> 当从 kernel 返回到 user 时，正常的 kernel_exit 会调用 trampoline 的 exit，tramp_exit 会重新将 kernel mapping 换成是 trampoline

## 2.2 KPTI 开销

​        KPTI引入了性能开销，因为进入内核和退出内核都需要额外的页表切换。通过加载 CR3 寄存器来完成切换，这大约需要300个周期。

同时由于 CR3 寄存器变化，TLB 将会被刷新，性能将进一步受到影响。有许多关于KPTI开销评估的报告，这些报告显示 KPTI 可能导致显著的性能成本(高达30%)，特别是在系统调用和中断严重的工作负载中。

​        在支持 PCID(进程上下文标识符)特性的处理器上，可以避免 TLB 刷新，并可以减少 KPTI 的性能开销。PCID 是页表的 12位 标记，保存为 TLB 条目的一部分。对于两个带有不同标记的页表，它们的 TLB 表项可以在 CPU 中共存，在两个页表之间切换时不需要TLB刷新。现有报告显示，启用 PCID 后，KPTI PostgreSQL 在 Intel Skylake 处理器上的只读测试开销从 17-23% 减少到 7-17%。然而，Linux 直到 2017 年 11 月 12 日发布的版本 4.14 才支持 PCID。

## 2.3 gPT, EPT and TLB

​        在本机环境中，PT（page table）用于将虚拟地址转换为物理地址。在虚拟化环境中，以非 root 模式运行的客户机 VM（virtual machine）仅通过 gPT（guest page table）控制其 GVA（客户机虚拟地址）到 GPA（客户机物理地址）的映射。hypervisor 以 root 模式运行，通过 hPT（主机页表）控制每个 VM 的 GPA 到 HPA（主机物理地址）的映射，hPT 在 Intel platform 1 上称为 EPT（扩展页表）。

​        图 2 显示了在具有 4 级 gPT 和 EPT 的 x86-64 机器上从 GVA 到 HPA 的转换过程。

<img src="/Users/li/Library/Application Support/typora-user-images/image-20210510211321062.png" alt="image-20210510211321062" style="zoom:50%;" />

guest CR3 的值和 gPT 内的地址是 GPAs，而 EPTP 的值和 EPT 内的地址是 HPAs。当 CPU 遍历 gPT 时，需要将所需 gPT 页的所有 GPA 通过 EPT 转换为 HPA。为了最小化页遍历期间的内存占用，处理器在虚拟化环境中有两种类型的 TLB:  EPT-TLB 和组合 TLB。EPT-TLB 用于加速 GPA 到 HPA 的翻译，而组合 tlb 存储 GVA 到 HPA 的翻译条目。

## 2.4 EPTP 与 VMFUNC 切换

​        VMFUNC 是一个“英特尔硬件虚拟化”扩展，它为以非 root 模式运行的 gust 虚拟机提供虚拟机函数，以便在不使用 VMExit 的情况下直接调用。目前，VMFUNC 只提供了一个 VM 函数，名为 “eptp switching”，它允许 gust VM中的软件（无论是内核模式还是用户模式）加载一个新的 EPTP（EPT指针）。gust 只能从 hypervisor 配置的有效 EPTP 值列表中切换到 EPEP。从 Haswell 架构开始，所有 Intel CPU 都支持 EPTP 切换。

​        **EPTP切换性能：**我们比较了加载 CR3 和 EPTP 切换的延迟。在启用 PCID 的情况下，将相同的值写入客户机 VM 中的 CR3 大约需要 300 个周期。而“ EPTP 切换”大约需要 160 个周期(两个不同的 EPTP 值，但有相同的映射)。

​		**EPTP 切换 TLB 行为：**我们进一步测试了 EPTP 交换的 TLB 行为，并发现 CPU 如何在 TLB 中构造不同 EPTP 的地址映射。使用一个 EPT 执行的操作不会影响其他 EPT，表 1 显示了测试结果。在表中，“两个 EPT 的 TLB 都无效，然后填充 EPT-0 的 TLB ”，这意味着我们首先在 EPT-0 和 EPT-1 中都调用 invlpg 指令（用于刷新 TLB），然后访问 EPT-0 的目标内存。之后，我们再次在 EPT-0 和 EPT-1 中访问目标内存，并测试访问延迟。 结果意味着 EPT-0 被填充，而 EPT-1 没有被填充。我们还测试了在一个 EPT 中调用  flush TLB 操作（write CR3 和 invlpg）是否会影响另一个 EPT 的 TLB。我们发现它们都刷新了其他 EPT 的 TLB。

<img src="/Users/li/Library/Application Support/typora-user-images/image-20210511131636943.png" alt="image-20210511131636943" style="zoom:50%;" />

# 3 系统概述

EPTI 有三个目标：

- Goal-1：达到与KPTI相同的安全级别。
- Goal-2：支持对未打补丁的虚拟机进行无缝保护。
- Goal-3：获得比KPTI更好的性能。

​        我们首先为每个 gust VM 构造两个 EPT：用于内核的 EPTk 和用于用户的 EPTu。EPTk 的映射与原始 EPT 相同（但是具有不同的权限，这将在以后介绍），因此内核将像以前一样运行。当用户应用程序运行时，我们需要确保它们不能访问(甚至推测地)内核地址空间中的任何数据。

​        一种直观的方法是删除 EPTu 中客户机内核使用的页面的所有 HPA 映射，这样当用户进程运行时，就不会映射所有内核页面。但是，这个解决方案不能工作，因为通常 Linux 内核会将整个 GPA 映射到它的 GVA 空间，这被称为直接映射，如图 3 所示。

<img src="/Users/li/Library/Application Support/typora-user-images/image-20210511132035155.png" alt="image-20210511132035155" style="zoom:50%;" />

这意味着我们必须从 EPT 中移除所有的 GPA 映射，这也将禁用用户进程的执行。

​        相反，我们只是重新映射了所有 gPT 的页面，这些页面将 EPTu 中的内核空间映射到了一个零位页面，如图 3 所示。因此，一旦用户进程试图使用其 GVA 访问内核地址，GVA 将永远不会被转换为 GPA，因为 CPU 无法在 gPT 中找到对应的映射(参见图 2 的左侧)。安全保证与 KPTI (**Goal-1**) 相同。

​        接下来，我们需要找到一种方法在适当的时间切换 EPT。当用户进程进入内核时，处理器应立即通过 VMFUNC 切换到 EPTk。在内核返回用户进程之前，它还会切换到 EPTu。在 Linux 内核中，入口点和出口点是有限的。入口点可以通过 IDT(中断描述符表)和一些特定的 MSRs(特定于模型的寄存器)来定位。出口点必须包含特定的指令（例如 sysretq）。因此，我们使用二进制工具即时重写内核代码，以在入口点和出口点插入两条 trampoline 代码，这些代码主要用于进行 EPT 切换。利用这种方法，可以将 EPTI 与实时迁移一起使用，从而无缝地保护客户机，而无需重新启动它。更多细节将在 4.3 节(**Goal-2**)中描述。

> 这里的 trampoline 代码和 KPTI 中使用的 trampoline 代码同理

​        为了在 EPTu 中取消映射内核空间，我们需要跟踪用于映射内核空间的 gPT 页面，并将它们在 EPTu 中清零。EPTI 跟踪 gL3 页面（用于翻译内核 GVA）并将其归零（详细信息见第 4 节）。我们将在第 5 节中进一步介绍我们的优化，以减少 vmexit 的数量并获得更好的性能(**Goal-3**)。

​        **挑战：**上述设计在安全性和性能方面仍存在许多挑战。例如，由于允许用户进程调用 VMFUNC，恶意进程可能会在发出 Meltdown 攻击之前尝试切换到 EPTk。我们将在接下来的文章中详细描述我们的设计，并提出这些挑战的解决方案。

# 4 EPT 隔离设计

​        在这一节中，我们介绍了 EPTI 的基本设计。首先，我们需要构造 EPTu，它删除了所有的内核地址映射。然后，我们介绍了如何跟踪内核 gPT 页面并添加用于 EPT 切换的 trampoline 代码的基本方法。最后我们构造了 EPTk，使恶意用户无法切换到它。

## 4.1 Zeroing gPT for Kernel Space in EPTu

​        <u>我们通过置零 EPTu 中用于地址转换的 gPT 页</u>来消除用户模式下内核地址空间的所有 GVA-to-GPA 映射。如上图所示，要使 gPT 页归零，我们将其重新映射到 EPTu 中的一个新的归零物理页。在使用 48 位虚拟地址的 64 位 Linux 内核中，有 4 个页面级别(从 gL4 到 gL1)。由于每个进程有不同的 gL4 页面，为了最小化对 EPTu 的修改，我们只将用于内核地址转换的 gL3 页面调零( gPT 结构如图 5 所示)。

<img src="/Users/li/Library/Application Support/typora-user-images/image-20210512131519700.png" alt="image-20210512131519700" style="zoom:50%;" />

​        将 EPTu 中内核空间的 gL3 页面置零后，由于目标 GVA 未被映射，从用户模式访问内核内存会触发 guest 页面错误(虽然 Meltdown 攻击可以绕过权限检查，但无法访问未映射的页面)。由于内核运行在 EPTk 中，它永远不能填满 EPTu 中归零的 gL3 页面，攻击者的用户进程永远不能访问内核内存(甚至推测性地)。

## 4.2 跟踪 EPTk 中的 gPT 页面

​        为了使在 EPTu 中映射内核空间的所有 gL3 页归零，EPTI 首先需要跟踪内核的所有 gL3 页。具体来说，通过将 VMCS（虚拟机控制结构）中的 `CR3_LOAD_EXITING` 位置 1，当 gust 内核更改 CR3 时，它将陷入 hypervisor，hypervisor 将遍历 gPT 以获得用于内核空间映射的所有 gL3 页面的列表。同时，gust 内核可以分配新的 gL3 页面并将其添加到 gL4。为了跟踪新的内核 gL3 页面，所有 gL4 页面将在 EPTk 中映射为只读，因此，每次 gust 内核将新的 gL3 页面添加到 gL4 时，它将陷入 hypervisor 以更新受监视的 gL3 页面列表。我们将在第 5 节中介绍跟踪的一些优化。

## 4.3 Trampoline for EPT Switching

```C
// Listing 1 EPT switching to EPTk
1: SWITCH_EPT_K:
2: 	SAVE_RAX_RCX
3:  movq $0 , %rax
4:  movq $0 , %rcx
5:  vmfunc
6:  RESTORE_RAX_RCX
```

​        trampoline 代码包含了 EPT 切换指令。Listing 1 显示了从 EPTu 切换到 EPTk 的汇编代码。%rax 和 %rcx 包含了 VMFUNC 索引和传递给 VMFUNC 的参数。第 3 行表示调用第一个 VMFUNC 函数(EPTP switching，index 0)，第 4 行表示切换到 EPT-0。%rax 和 %rcx 都是调用者保存的，因此需要在 trampoline 中保存和恢复这些值。`SWITCH_EPT_U` 的过程类似，但方向相反。因为 trampoline 代码用于在 EPTk 和 EPTu 之间切换，所以需要在两个 EPT 中都调用它。我们需要确保：（1）这两个 EPT 中都可以执行 trampoline，并且（2）有一个合适的地方来保存调用者保存的寄存器。

​		**将 trampoline 映射为两个 EPT 中的可执行文件：**<u>在 EPTu 中，只有带有 trampoline 代码的一页会映射到内核空间中</u>。为了确保这一点，EPTI 将所有 gPT 页( gL4 除外)重新映射到新的 host 物理页，这用于转换 trampoline 的 GVA。然后这些页面的所有条目都设置为 0，除了用于映射 trampoline 的条目(如图 4 所示)。

<img src="/Users/li/Library/Application Support/typora-user-images/image-20210512134616913.png" alt="image-20210512134616913" style="zoom:50%;" />

gust IDT 的条目和系统调用条目 MSR (IA32 LSTAR) 将被更改为指向 trampoline 代码。在 EPTk 中，EPTI 将 trampoline 代码插入到 guest 内核的直接映射区域的末尾，并重新写入 kernel 的二进制，将退出点更改为将控制传递给 trampoline 的 jmp 指令。

> 这相当于添加 trampoline，并且重新映射客户机的所有 page

​		**保存调用者保存的寄存器：**因为 VMFUNC 不会通过硬件保存任何寄存器（这使它变得很快），所以 trampoline 代码在保存它们之前不能使用任何寄存器。保存这些调用者保存的寄存器的一个挑战是支持多核。对于单 CPU 内核，可以将 `％rax` 和 `％rcx` 的值保存到带有固定地址的内存页中。<u>但是，对于多核，一个核可能会覆盖另一核的已保存寄存器值，因为它们写入的是同一地址。</u>

​		Linux 通过使用 gs-based 的 per-cpu value 解决了这个问题。在系统引导期间，它为每个核心分配一个 `per-cpu` 内存区域。这些区域的基址在进入内核后(通过 swapgs 指令)通过不同核心的 gs 寄存器记录。per-cpu value 的以下访问由 `％gs: index` 执行。EPTI 无法利用此方法，因为：（1）需要了解内核的某些特定语义，以及（2）需要将内核的 per-cpu 区域映射到 EPTu。

​		EPTI 提供了 per-vCPU 内存页来保存和恢复调用方保存的寄存器。具体来说，一个内存页面被映射到 EPTu 和 EPTk 中的内核空间中。要启用来自多个内核的并发访问，该页面将被映射到不同 vCPU 的不同物理页面，以便当一个 vCPU 保存 ％rax 和 ％rcx 时，会将值写入其自己的页面。在我们的实现中，我们修改 gPT 以将这个页面映射到一个未使用的 GPA (例如，客户 DRAM 范围之外的 GPA)。在不同 vCPU 的 EPT 中，我们将此 GPA 映射到不同的 HPA。 在一个 vCPU 的 EPTk 和 EPTu 中，它都映射到相同的 HPA。

​		**无缝的保护：**结合实时迁移，EPTI 可以无缝保护 gust VM，而无需重新启动它。云提供商可以迁移掉所有 vm，更新主机 hypervisor 以启用 EPTI，并将所有 vm 迁移回来。为了在 EPTI 上恢复一个 VM，我们需要：（1）将 trampoline 映射到客户机;（2）重写中断和系统调用的条目（存储在 IDT 和 MSR 中），以及退出点（包含特定的指令，如 sysretq），以跳转到 trampoline;（3）启用 gPT 和 gust EPTP 切换的陷入。

## 4.4 恶意 EPT 切换

​		以上设计基于这样一个假设，即只有内核可以切换 EPT。 不幸的是，VMFUNC 指令可以在 guest 内核模式或 guest 用户模式下调用，这使攻击者可以通过 VMFUNC 恶意切换到 EPTk，发出Meltdown攻击并切换回 EPTu。 为了防御这种攻击，EPTI 需要使 EPTk 对用户进程无用。

​		通过执行真正的 Meltdown 攻击，我们发现尽管 Meltdown 可以在没有访问权限的情况下读取内存，但即使在重新执行命令的情况下，也无法在没有可执行权限的情况下获取代码。 基于此观察，我们将所有用户内存映射为 EPTk 中的 execute-never。 因此，一旦用户恶意切换到 EPTk，其所有代码将不可执行。

​		具体来说，EPTI 只将 guest 物理内存(包括内核的代码和内核模块)映射为 EPTk 中的可执行文件，而所有其他 guest 物理内存都映射为 “execute-never”。内核代码是在系统引导过程中加载的，在执行过程中不会更改，EPTI 可以通过搜索 gPT 来检测所有相应的 GPA。内核模块在运行时动态加载/删除，EPTI 需要监视用于它们的所有 guest 物理页面。这是通过在 gPT 页面上捕获所有写入操作来完成的，gPT 页转换了内核模块的 GVA 到 GPA 的映射。由于 Linux 内核为内核模块保留了一个固定的 GVA 区域，因此捕获对相应 gPT 页面的修改只会影响安装/删除内核模块的性能。

# 5 优化

​		如上一节所述，EPTI 需要捕获客户机 vm 中的 load-CR3 操作和 write-gL4 操作。这些捕获方法有三个性能问题：

- **无用的 load-CR3 捕获：**EPTI 捕获客户机 VM 的 load-CR3 操作以获得所有 gPTs。事实上，EPTI 只需要捕获新的 gPTs，但是大多数 load- cr3 操作只是加载旧的 gPTs，这导致了许多无用的 vmexit。

- **Access/Dirty bits update：**为了捕获 gPT 页的修改，EPTI 在 EPTk 中将其标记为只读。然而，对于每个内存访问（包括读、写和取），<u>CPU 将更新用于转换目标 GVA 的 gPT 条目中的 A/D 位</u>（访问/脏位），即使 A/D 位已经由先前的操作设置。因此，每当内核访问其任何数据时，都会触发 EPT 冲突，这将导致大量 VMExit。

- **write-gPT 的其他捕获：**在第 4.2 节中，EPTI 捕获所有 write-gL4 操作，以跟踪内核空间映射中所有启用的 gL3。但是，每个进程都有一个 gL4 页面，这意味着 EPTI 需要捕获数千个 gL4 页面。由于每个进程的内核地址映射都相同，因此可以优化捕获所有这些 gL4 的过程。

  在本节中，我们将给出几个优化来解决所有这些性能问题。

## 5.1 选择性跟踪 guest CR3

​		EPTI 利用硬件功能来减少由捕获旧 CR3 引起的 VMExit 数量。英特尔在 `VMCS` 中提供 4 个 `CR3_TARGET_VALUE` 字段。如果 guest 虚拟机中的 load-CR3 的源操作数与这些值之一匹配，则不会导致 VMExit。我们将 CR3 值写入 `CR3_TARGET_VALUE`：1）导致每秒超过阈值 A 的 VMExits 或 2）完全导致超过阈值 B VMExits（VMM 可以配置 A 和 B）。

> 当某一个 CR3 重新写入导致的 VMExit 次数达到阈值 A 或 B 的时候，说明这个 gPT 被经常使用，就把它写入 `CR3_TARGET_VALUE` 中，从而不会引发 VMExit

## 5.2 设置 gPT 的 Access/Dirty-Bit

​		为了在 CPU 设置 A/D 位时消除 VMExit，我们需要允许 CPU 编写 gPT，同时不允许内核这样做。我们发现它们的访问路径不同：内核通过其 GVA（同时使用 gPT 和 EPT）写入 gPT，而 CPU 通过其 GPA（仅使用 EPT）写入 gPT。因此，EPTI 在 EPT 中映射具有写许可权的 gPT 页面，以允许 CPU 更新 A/D 位。为了阻止内核修改 gPT 页，我们将该页的 GVA 重定向到一个新的 GPA，该 GPA 在 EPTk 中映射为只读。这是由：

1. 修改用于目标 gPT 页面的 GVA 到 GPA 转换的 gL1 条目，并将 gPT 页面重新映射到新的 GPA；
2. 将新 GPA 以只读方式映射到原始 HPA，其中包含目标 gPT 页面。

因此，只有内核对 gPT 页面的写访问权才会触发 VMExit。

## 5.3 捕获 gL3 page 而不是 gL4 page

​		根据以下观察，我们采用了另一种优化方法：

- *大多数内核虚拟地址区域从不更改。*Linux 内核为不同的用途保留了内存区域，并且它从不更改大多数这些区域的映射（例如，直接映射区域永远不会更改）。
- *每个 gL3 页面可以转换一个较大的虚拟空间（512GB）。*在 guest 中，内核几乎不可能分配这么多虚拟内存，因此内核 gL3 页的数量很少改变。
- 在 Linux 内核实现中，仅当现有 gL3 页面的所有条目都在使用中，或者连续的自由条目不足时，内核才会创建新的 gL3 页面。

​		基于以上观察，默认情况下，EPTI 直接捕获 gL3 页面的内核修改。 使用 gL3 的最后一个条目时，这意味着内核可以在以后分配新的 gL3 页面，EPTI 开始捕获 load-CR3 和 write-gL4，直到分配了新的 gL3 页面。通过这种优化，EPTI 几乎不需要捕获 load-CR3 和 write-gL4 的操作，这将减少大多数（如果不是全部）VMExits。