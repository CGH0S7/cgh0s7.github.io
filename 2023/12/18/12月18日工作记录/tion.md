---
title: 面向竞赛环境的Linux系统性能调优实践
date: 2026-01-18 12:14:04
tags:  [技术, 系统优化, HPC, Linux内核]
categories: [技术分享，HPC]
---

最初，我的想法是现在大伙都有AI工具了码力差距将不再像过去那么明显，这种情况下系统调优一定是有实战价值的。尤其是在面对极端计算任务时，操作系统内核的调度策略和中断处理方式对性能必然有着直接且显著的影响。为了在计算密集型任务（如本次天体物理模拟）中获得极致的性能与确定性，我作为运维必须尝试对Linux内核进行深度定制与隔离。

本文将详细记录我在Intel Sapphire Rapids平台（Dual Xeon 8480P+）上进行的系统调优探索。通过内核参数隔离、用户态调度器（sched_ext）以及IRQ亲和性管理，我成功在AMSS-NCKU双黑洞模拟算例中取得了肉眼可见的性能收益。

## 1. 测试环境与硬件拓扑

本次调优的硬件平台具备极高的并行计算能力，但NUMA架构带来的延迟敏感性也对系统配置提出了挑战：

* **CPU**: 2x Intel Xeon Platinum 8480+ (Sapphire Rapids)
* **核心配置**: 单路56核心，双路共112物理核心。
* **超线程 (Hyper-Threading)**: **Disabled**（为了避免逻辑核争抢浮点单元及缓存资源，确保计算流水线的独占性）。
* **GPU**: 无（纯CPU计算任务）。

针对该拓扑，我制定了**“核心隔离（Core Isolation）”**策略：将系统中断、管理进程及内核Housekeeping任务通过`cpuset`和内核参数限制在特定的少数核心上（Housekeeping Cores），将绝大多数核心完全腾空，仅运行计算任务（Isolation Cores）。

* **Housekeeping Cores**: 0-3 (Socket 0), 56-59 (Socket 1) — 共8核。
* **Isolation Cores**: 4-55 (Socket 0), 60-111 (Socket 1) — 共104核，用于纯计算。

## 2. 内核启动参数（Kernel Command Line）深度解析

首先需要确定的是这次我将选择Rockylinux 10作为操作系统基础，内核版本为6.12是首个支持sched_ext调度器的稳定版本，并且Rockylinux 10首次完全基于x86-64-v3构建，相比其他发行版有着必然的性能优势，而少数比它性能更好的发行版（CachyOS和已经逝世的Clearlinux）又没它稳定还不受nvidia ofed驱动支持，考虑到这些因素我不会多看其他任何发行版一秒。（现在都是2026年了坚决抵制CentOS 7/8或Rockylinux 8信徒遗老，这么老的内核挂个exFAT硬盘传东西都费劲）。

为了实现我预期的隔离策略，我使用grubby在GRUB配置中启用了以下内核参数。这些参数共同构成了一个低延迟、低抖动的运行环境。

```bash
transparent_hugepage=never iommu=pt mitigations=off \
nohz_full=4-55,60-111 rcu_nocbs=4-55,60-111 \
isolcpus=managed_irq,4-55,60-111
```

### 2.1 内存与IO子系统优化

* **`transparent_hugepage=never`**: 禁用透明大页（THP）。虽然THP能减少TLB Miss，但在高强度的随机内存访问或频繁申请释放场景下，THP的后台合并线程（khugepaged）会引入不可预测的延迟抖动（Jitter）。
* **`iommu=pt`**: 将IOMMU设置为Pass-Through模式。这允许设备在DMA传输时直接使用物理地址，绕过IOMMU的地址转换层，显著降低了IO密集型操作的CPU开销，同时保留了IOMMU的基本功能。

### 2.2 安全缓解措施禁用

* **`mitigations=off`**(让CPU回归其原本的样子): 该参数关闭了针对Spectre、Meltdown等侧信道攻击的所有CPU漏洞缓解措施。这些缓解措施通常涉及频繁的页表隔离（PTI）和间接分支预测限制，在旧时代这会带来非常明显的性能损耗（通常在15%-25%之间），但是在今天能够提供的性能提升根据之前不知道在哪里看到的Intel测评已经不到5%了。在追求极致速度的物理隔离环境中，我还是要物尽其用，认为关闭它们是提升IPC（Instructions Per Cycle）的必要手段。

### 2.3 核心隔离与时钟中断消除 (核心措施)

* **`nohz_full=4-55,60-111`**: 启用Full Dynticks模式。在指定的核心上，如果运行队列中只有一个任务（即我们的计算进程），内核将停止发送调度时钟中断（Tick）。这消除了周期性的上下文切换干扰，实现了“无干扰（Tickless）”执行。
* **`rcu_nocbs=4-55,60-111`**: 将指定核心上的RCU（Read-Copy-Update）回调函数卸载到非隔离核心（Housekeeping cores）的线程中执行。这防止了RCU软中断打断关键计算路径。
* **`isolcpus=managed_irq,4-55,60-111`**: 将指定核心从内核的通用SMP均衡调度器中移除，确保普通进程不会被随机调度到这些核心上。
* **`managed_irq`** 标志进一步指示内核尽量避免将受管中断（Managed Interrupts）分配给这些核心，为后续的用户态中断控制打下基础。

## 3. 用户态环境与调度优化

除了内核层面的静态配置，运行时的动态调优同样关键。

### 3.1 基础系统调优 (`tuned`)

```bash
tuned-adm profile hpc-compute
```

启用RedHat系的`hpc-compute`配置文件。该Profile会自动调整虚拟内存参数（如`vm.dirty_ratio`）、网络栈缓冲区大小以及CPU的电源管理策略（强制最大C-State为C1或C0），确保CPU始终处于高性能状态，避免频率升降带来的延迟。

### 3.2 scx调度器：sched_ext (`scx_tickless`)

```bash
/usr/local/bin/scx_tickless -m 0x0F0000000000000F
```

利用较新的Linux内核特性（BPF-based extensible scheduler class），我引入了`scx_tickless`调度器。

* 该调度器由英伟达工程师设计，通过eBPF程序接管调度决策，相比完全公平调度器（CFS），它能更激进地维持计算核心的Tickless状态。
* 参数 `-m 0x0F0000000000000F` 是一个掩码配置，精确指定了Housekeeping核心的拓扑位置，确保调度器仅在管理核心上处理必要的调度逻辑，从而保护计算核心的“静默”状态。

### 3.3 中断亲和性管理 (`irqbalance`)

为了防止网卡、磁盘控制器的硬中断打断计算流水线，必须严格限制中断响应的核心范围。

```bash
# 配置文件/etc/sysconfig/irqbalance中设置
IRQBALANCE_BANNED_CPUS="FFFFFFFFFFFFF0FFFFFFFFFFFFF0"
```

这是一个64位/128位的十六进制掩码。该配置显式**禁止**了所有隔离核心（即计算核心）参与中断负载均衡。结果是所有硬件中断都被强行“钉”在了Housekeeping Cores（Socket 0的0-3和Socket 1的56-59）上，保证了计算核心100%的CPU时间片用于业务逻辑。

### 3.4 进程绑核 (`taskset`)

最后，在启动计算任务时，最好显式指定亲和性不能光依赖scx_tickless调度，防止跨NUMA节点的内存访问惩罚。

```makefile_and_run.py
# 添加一个启动脚本变量
NUMACTL_CPU_BIND = "taskset -c 4-55,60-111"
```

结合`isolcpus`，这确保了计算线程被牢牢锁定在无干扰的核心上，且每个线程独占物理核L1/L2缓存。

## 4. 性能验证与结论

**测试用例**: AMSS-NCKU双黑洞模拟（基于广义相对论数值解法）。
**评估指标**: 单个时间步（Time Step）的计算耗时。

在应用上述全套优化措施后，系统表现出极高的确定性。对比默认配置（Default CFS + On-Demand Power + Default IRQ Balance）：

1. **稳定性提升**: 计算过程中的抖动（Jitter）几乎消失，btop监控可以看到各核心负载曲线呈现完美的平直线。
2. **性能收益**:

* 在计算前期，单步时间步稳定降低 **0.1 - 0.2秒**。
* 随着模拟进入后期（网格加密或相互作用增强），计算密度增加，该优化带来的累计收益更为显著。
* 48核心运行总时间预计缩短约 **800秒**。

### 总结

过去参加比赛就没有一次是能够在限定功耗下启用所有核心不降频全力跑程序的，我不禁思考既然用不完何不将单核计算性能调优做到极致，牺牲一些核心处理杂活不参与计算，剩余的核心全力以赴跑程序不要受干扰最好也不要降频，这就是我的大致思路。

通过`nohz_full`与`isolcpus`实现核心隔离，配合`mitigations=off`释放硬件潜能，并利用`scx_tickless`进行用户态调度增强，我按照设想将这台Xeon 8480+服务器改造成了一个OS Noise几乎消除的专用计算平台。后续需要关注的问题是迁移服务器后如何快速重现这些调优配置，或者干脆设计动态调控脚本来根据硬件信息调整内核参数。另外还有多机运行也没有进行测试，未来会关注在MPI环境下验证这些调优措施的适用性。

---

### 赛后更新

散了吧孩子们，H100/H200已经杀死比赛了:(
