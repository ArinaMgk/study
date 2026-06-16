每核跳板栈 + 统一内核页表 + 线程物理内核栈 方案

目标

解决当前 x8600 内核中 `_GUI_FREETYPE` / 磁盘代理读路径被调度放大的问题，
同时避免“每个活线程永久占一段 kernel stack VA”带来的地址空间压力。

本方案放弃“每次调度 remap fixed stack window”，也放弃“每线程永久 kernel stack VA”。

改为：

1. 每个 core 固定一个很小的 ring0 跳板栈
2. `TSS.ESP0/RSP0` 始终固定指向这根跳板栈
3. 从用户态进入内核时，先落在跳板栈
4. 跳板代码切换到统一内核页表
5. 再切到当前线程自己的真实内核栈那块内存
6. 后续大部分内核逻辑都在“统一内核页表 + 线程真实内核栈”上执行


零、第一步：先固定三种栈的语义和虚拟地址

这一节是整个方案的第一步，也是后续所有实现的前提。

在开始任何汇编、页表、调度器改造之前，必须先把当前 x8600 内核中的几类“栈”语义彻底分开。

以后代码和注释都按下面三类理解，不再混用。

先直接给出本方案第一步采用的虚拟地址布局。

这里**不新开辟额外的 PERCORE 跳板栈虚拟区**，
**不新增 transition stack 专用 VA 区**，
而是**原样复用**你当前
`_Mapping_Core_Stack(Paging& paging)` 里已经映射的那一组地址。

这一点必须固定：

1. 第一阶段只复用现有 `_Mapping_Core_Stack()` 地址
2. 不讨论另起一套 PERCORE 跳板栈空间
3. 不把“语义重定义”误写成“地址重划分”

x8632:

1. per-core transition stack / return workspace
   - 直接复用：
     `vaddr = 0xFFFFF000u - cpu_i * 0x1000u`
   - 即：
     - core0: `0xFFFFF000..0xFFFFFFFF`
     - core1: `0xFFFFE000..0xFFFFEFFF`
     - core2: `0xFFFFD000..0xFFFFDFFF`
   - 对应内存来源：
     `ring3_iret_stacks[cpu_i]`

2. per-thread real kernel stack
   - 不单独分配 permanent VA 大区
   - 继续使用 `stack_levladdr` 对应的那块真实内核栈内存
   - 统一内核页表负责让这块内存可直接作为切栈目标使用

3. 说明
   - `ring3_iret_stacks[cpu_i]` 这块以后不再只视作“iret 栈”
   - 统一重新定义为：
     `per-core transition stack`
   - 它同时承担：
     - 用户态进入内核的入口跳板栈
     - 内核返回用户态前的整理栈

x8664:

1. per-core transition stack / return workspace
   - 直接复用：
     `0x0000FFFFFFFFF000ull - cpu_i * 0x1000ull`
   - 即：
     - cpu0: `0x0000FFFFFFFFF000ull`
     - cpu1: `0x0000FFFFFFFFE000ull`
   - 当前代码只先映射了 cpu0（因为 x8664 现在只支持单核）
   - 本文第一阶段只承认当前这条已存在映射
   - 对应内存来源：
     `higher_stacks[cpu_i]`

2. per-thread real kernel stack
   - 不单独分配 permanent VA 大区
   - 继续使用 `stack_levladdr` 对应的那块真实内核栈内存
   - 统一内核页表负责让这块内存可直接作为切栈目标使用

3. 说明
   - x8664 当前 `_Mapping_Core_Stack()` 只映射了 cpu0
   - 这是因为 x8664 当前只支持单核，不是本文第一阶段要处理的问题

总原则：

1. `transition stack` / `return workspace`
   - **复用现有 `_Mapping_Core_Stack()` 的虚拟地址**
   - **不再另起一套 PERCORE 跳板栈 VA**

2. `real kernel stack`
   - 继续来自线程自己的 `stack_levladdr`
   - 不再走 per-thread permanent slot VA

1. per-core transition stack

定义：

- 每个 core 一根固定的小栈
- `TSS.ESP0` / `TSS.RSP0` 固定指向它
- 仅用于 ring3 -> ring0 入口过渡，和 ring0 -> ring3 返回整理

允许承担的职责：

- 接住 CPU 从用户态进入内核时自动切过来的第一落点
- 保存最少量早期现场
- 读取 `PERCORE`
- 找到 `current_thread`
- 切换到统一内核页表
- 切换到线程真实 kernel stack
- 把此前保存在 transition stack 上的必要上下文转移到线程真实 kernel stack
- 返回用户态前整理 `iret/iretq` 所需现场

禁止承担的职责：

- 不作为线程正常内核执行栈
- 不承载长时间运行的 C 代码
- 不允许在线程可调度状态下长期保留上下文

虚拟地址规定：

x8632:

- 直接复用 `_Mapping_Core_Stack()`：
  `0xFFFFF000u - cpuid * 0x1000u`
- 每 core 目前就是 1 页
- `TSS.ESP0` 固定指向该页顶部附近
- 建议落点：
  `0xFFFFF000u - cpuid * 0x1000u + 0x1000u - 0x10u`

x8664:

- 直接复用 `_Mapping_Core_Stack()` 当前映射的地址
- `0x0000FFFFFFFFF000ull - cpuid * 0x1000ull`
- 当前已存在：
  - cpu0: `0x0000FFFFFFFFF000ull`
- 目标规则：
  - cpu1: `0x0000FFFFFFFFE000ull`
- `TSS.RSP0` 固定指向该页顶部附近
- 当前先只考虑这一个已存在的单核映射

2. per-thread real kernel stack

定义：

- 每个线程独立拥有的一块真实 ring0 栈内存
- 当前代码中主要对应 `ThreadBlock::stack_levladdr`

允许承担的职责：

- 线程在内核中的正常执行栈
- syscall / trap / 中断进入后，最终应落在这里保存完整上下文
- 线程阻塞、调度切换、恢复继续执行时依赖的正式内核栈

要求：

- 必须在“内核真正执行所用的 CR3”下稳定可访问
- 任何跨调度边界存活的内核上下文都必须最终落在这里

地址规定：

本方案不再给它单独划 permanent kernel stack region，
也不把某个固定高半区导出地址写成架构规则。

统一规定：

- `stack_levladdr` 视为线程真实内核栈那块内存的基址
- trampoline 在切到统一内核 CR3 后，
  直接以这块内存为目标切到线程真实 kernel stack 顶部
- 统一内核页表必须保证：
  这块内存在该 CR3 下可直接用作切栈目标

换句话说：

- 这条方案故意不再给每线程预留一段长期固定 slot VA
- 也不把 `mglb(...)` 一类高半区导出地址当作方案定义
- 方案真正承认的只有：
  `stack_levladdr` 是线程真实内核栈那块内存

3. return / iret frame workspace

定义：

- 返回用户态前用于拼装 `iret/iretq` 所需现场的短暂工作区

语义规定：

- 这个工作区可以与 per-core transition stack 复用
- 但它不是独立的线程运行栈
- 只是返回用户态前的过渡阶段

虚拟地址规定：

- 与 per-core transition stack 完全复用
- 也就是直接复用 `_Mapping_Core_Stack()` 当前那组地址
- 不再额外设第二块独立 VA 区

因此，在本方案中：

- `transition stack` 和 `return workspace` 可以复用
- `thread real kernel stack` 不能与它们混为一谈

一句话定性：

- transition stack = 每核固定、只做过渡
- real kernel stack = 每线程独立、承载真正执行
- return workspace = 过渡阶段，可与 transition stack 复用

一句话定址：

- transition stack / return workspace:
  直接复用 `_Mapping_Core_Stack()` 当前映射的那组地址
- real kernel stack:
  以 `stack_levladdr` 这块真实内核栈内存为准

从这一节开始，后文出现“跳板栈”“入口栈”“返回栈”等词时，统一按上面定义解释。


一、为什么选这条路线

当前已有两个事实：

1. 你现在已经是一线程一块真实内核栈内存
   - `stack_levladdr` 是每线程单独分配的 ring0 栈内存

2. 当前最重的不是“有线程栈”本身，而是：
   - 每次调度都把 `GetCoreRingStackBase(cpuid)` 对应 fixed window
   - remap 到不同线程的 `stack_levladdr`
   - 同时改多份页表
   - 再按页刷新整段窗口

如果继续走“每线程长期 kernel stack VA”方案，虽然能去掉 rebind，
但会遇到你已经指出的问题：

- 活线程太多时，每线程永久占 VA slot，规模不优雅

所以更合适的办法是：

- 保留“每线程一块真实内核栈内存”
- 去掉“每线程永久 VA”
- 改用“每核固定小跳板栈 + 统一内核页表”

这样线程规模主要消耗：

- 线程真实内核栈内存
- 调度结构

而不会线性吞掉一整片 per-thread permanent VA 区。


二、核心思想

把“入口栈”和“运行栈”彻底分开。

1. 入口栈

- 每 core 一根
- 很小
- 仅用于用户态陷入内核后的最早期过渡
- `TSS.ESP0/RSP0` 固定指向它

2. 运行栈

- 每线程一块真实内核栈内存
- 即现有 `stack_levladdr`
- 不再要求它对每个进程 CR3 都有长期固定 VA

3. 内核页表

- 单独准备一套“统一内核页表”
- 该页表必须能稳定访问：
  - 内核代码/数据
  - 所有线程的真实内核栈内存
  - per-core 数据
  - GDT/IDT/TSS
  - MMIO / APIC / 中断控制器
  - 文件系统 / 设备服务运行所需数据

4. 进入内核后的阶段划分

阶段 A：用户 CR3 + per-core 跳板栈
阶段 B：统一内核 CR3 + 当前线程真实 kernel stack

也就是说：

- `ESP0` 只负责“从 ring3 进 ring0 时先别死”
- 真正的内核执行不再依赖 `ESP0` 对应栈长期承载线程


三、当前代码中哪些东西保留，哪些东西废弃

保留：

1. 每线程 `stack_levladdr`
2. 每 core `GetCoreRingStackBase(cpuid)` 这一类固定高地址概念
3. `TSS.ESP0/RSP0`
4. `current_thread`
5. `SwitchTaskContext`

废弃或降级：

1. `RebindThreadKernelStackWindow()`
   - 不再作为调度热路径的一部分

2. “fixed stack window 承担线程运行栈”
   - 只保留给入口跳板，不再承载正常线程执行期

3. “给每个线程注册 permanent kernel_stack_va”
   - 本方案不采用


四、地址空间模型

本方案不再要求“每个进程 CR3 中都看见所有线程 kernel stack”。

改成以下模型：

1. 用户进程 CR3
   - 看见本进程用户空间
   - 看见最小必要的内核高地址
   - 看见 per-core 跳板栈入口区
   - 不要求看见所有线程的真实内核栈内存

2. 统一内核 CR3
   - 主体采用物理内存恒等映射
   - 看见全部内核代码/数据
   - 看见全部线程真实内核栈内存
   - 看见全部核心内核服务对象
   - 看见 per-core 数据、GDT/IDT/TSS、设备 MMIO

2.1. “主体采用物理内存恒等映射” 的具体含义
   - 统一内核页表中的主工作区，不依赖 `mglb(...)` 这类额外高半区导出规则
   - 线程真实内核栈那块内存、内核对象、常用物理页，在这张页表里优先按“地址数值等于物理地址数值”的方式访问
   - trampoline 在切到统一内核 CR3 后，后续切运行栈时使用的就是这套恒等映射地址
   - 因此：
     - `TSS.ESP0/RSP0` 固定指向 per-core transition stack
     - 线程运行期的 `ESP/RSP` 则切到统一内核页表主体恒等映射下的线程真实内核栈地址

3. 线程真实运行时
   - 原则上在统一内核 CR3 上继续
   - 不再停留在用户进程 CR3 上跑完整个内核逻辑


五、跳板栈的职责

跳板栈必须极小、极简单。

允许做的事：

1. 保存最小硬件现场
2. 确认当前 core id
3. 取出 `current_thread`
4. 取出当前线程真实内核栈那块内存的顶部
5. 切到统一内核 CR3
6. 切 `ESP/RSP` 到线程真实 kernel stack
7. 跳转到常规内核入口继续执行

这里的“切 `ESP/RSP` 到线程真实 kernel stack”，
指的是：

- 切到统一内核 CR3 主体恒等映射下，
- 线程真实内核栈那块内存对应的恒等映射地址顶部

不是：

- 某个 `mglb(...)` 高半区导出地址
- 也不是裸物理地址概念下不经页表直接写入 `ESP/RSP`

不要在跳板栈里做的事：

1. 不做复杂文件系统逻辑
2. 不做页分配
3. 不做消息发送
4. 不长时间停留
5. 不做递归/深调用

一句话：

跳板栈只干“接住用户态 -> 切页表 -> 切线程栈”这件事。


六、统一内核页表的要求

统一内核页表必须成为“内核主执行环境”。

第一原则：

- 主体采用物理内存恒等映射
- 后续大部分内核执行默认依赖这套恒等映射工作
- 不是默认依赖用户进程 CR3
- 也不是默认依赖额外高半区导出地址

至少包含：

1. 内核 text / rodata / data / bss
2. 内核 heap / mempool
3. 所有 `ThreadBlock`
4. 所有 `ProcessBlock`
5. 所有线程的 `stack_levladdr`
6. per-core 数据 (`PERCORE`)
7. GDT / IDT / TSS
8. APIC / PIC / MMIO / 设备寄存器区
9. 文件系统 / VFS / inode / dentry / page cache
10. 驱动数据缓冲区

关键要求：

- 只要进入“统一内核页表阶段”，内核不应再依赖“当前用户进程 CR3 正好还映着什么”


七、线程真实 kernel stack 的访问方式

本方案依赖一个前提：

线程的 `stack_levladdr` 必须在统一内核页表里稳定可访问。

因此要做：

1. 创建线程时：
   - 继续分配 `stack_levladdr`
   - 同时把这段真实内核栈内存映射进统一内核页表

2. 线程退出时：
   - 从统一内核页表解除映射
   - 再释放这段真实内核栈内存

这里注意：

- `stack_levladdr` 目前语义偏“真实内核栈那块内存 / 物理侧”
- 本方案要求统一内核页表直接覆盖这块内存
- 这样 trampoline 切到统一内核 CR3 后，
  就可以直接把它当作线程真实 kernel stack 的基址来用


八、调度器如何改变

现在的高成本来自：

1. 调度时调用 `RebindThreadKernelStackWindow()`
2. 改多份页表
3. 刷整段窗口

本方案下，调度器改成：

1. 切线程时不 remap fixed window
2. 不按页刷新一整段 stack window
3. 只更新：
   - `current_thread`
   - 新线程上下文
   - 如有需要更新 per-core 记录

也就是说：

旧模型：
  Schedule() -> Rebind stack window -> SwitchTaskContext()

新模型：
  Schedule() -> 仅切上下文 -> SwitchTaskContext()

`TSS.ESP0/RSP0` 不再在普通调度里频繁跟着线程变化，
而是长期固定在 per-core 跳板栈顶部。


九、syscall / 中断入口如何改变

这是本方案最关键的改造点。

入口路径应改成两段。

第一段：硬件入口段

1. CPU 从 ring3 切到 ring0
2. 自动使用固定 `ESP0/RSP0`
3. 落到 per-core 跳板栈

第二段：软件跳板段

1. 保存最小上下文
2. 根据 `current_thread` 找到线程真实 kernel stack
3. 切到统一内核 CR3
4. 切到线程真实 kernel stack
5. 再进入普通 syscall / interrupt 分发逻辑

这个顺序非常重要：

- 先有栈
- 再切 CR3
- 再切线程真实栈
- 再跑复杂逻辑

不能反过来。


十、用户态指针访问策略

本方案的代价之一是：

进入统一内核页表后，
当前用户进程的虚拟地址不一定天然可见。

所以必须明确制度：

1. 内核主执行环境默认不直接解引用用户指针
2. 所有用户指针访问都必须通过显式桥接

桥接方式可以是：

1. 显式 copy
   - `copy_from_user`
   - `copy_to_user`

2. 临时映射用户页

3. 极少数场景临时切回用户 CR3 再访问
   - 不推荐作为主路径

最重要的是：

不能继续默认认为“当前在内核里，用户地址自然能看见”。


十一、同一进程多线程并发进入内核会不会踩栈

本方案下，不会因为“同一进程”而踩栈。

因为：

1. 每个 core 只共享自己的小跳板栈
   - 但每 core 一根，不同 core 不共享

2. 跳板阶段很短

3. 真正执行期切到的是：
   - 当前线程自己的真实内核栈那块内存

所以：

- 同一进程两个线程分别在两个 core 上进内核
- 会分别落到各自 core 的跳板栈
- 然后切到各自线程的真实 kernel stack
- 不会因为“同进程”而共栈


十二、相比方案 A 的优缺点

优点：

1. 不需要为每个活线程长期占一段 permanent kernel stack VA
2. 线程很多时，不直接被 per-thread permanent VA 数量限制
3. `ESP0/RSP0` 可以 per-core 固定
4. 调度器可彻底摆脱 `RebindThreadKernelStackWindow()`

缺点：

1. syscall / interrupt 入口更复杂
2. 统一内核页表必须设计得很稳
3. 用户指针访问必须全部显式桥接
4. 需要非常谨慎处理嵌套中断和异常路径


十三、最小实施顺序

阶段 1：固定每 core 跳板栈语义

1. 明确 `GetCoreRingStackBase(cpuid)` 不再是“线程运行栈窗口”
2. 它只作为：
   - entry / trampoline stack
3. `TSS.ESP0/RSP0` 始终固定指向它

阶段 2：准备统一内核页表

1. 明确一套专门的 kernel-only CR3
2. 确保其中能访问：
   - 全部线程 `stack_levladdr`
   - 全部核心内核对象
3. 先只读验证，不急着切

阶段 2 的最小产出不是“先写代码切过去”，
而是先列出一张“统一内核页表最小必须可见清单”，逐项核对当前映射。

第二步：统一内核页表最小必须可见清单

在开始修改 syscall / interrupt 入口之前，
必须先确认切到统一内核 CR3 后，下面这些对象仍然都能直接访问。

A. 入口过渡必需对象

1. per-core transition stack
   - 即 `_Mapping_Core_Stack()` 当前映射的那一页
   - x8632:
     `0xFFFFF000u - cpu_i * 0x1000u`
   - x8664:
     `0x0000FFFFFFFFF000ull - cpu_i * 0x1000ull`
   - 用途：
     - 用户态进入内核后的第一落点
     - 切到统一内核 CR3 后，若还未切线程真实 kernel stack，仍可能短暂继续使用
     - 返回用户态前的 `iret/iretq` 整理区

2. `PERCORE`
   - 必须在统一内核 CR3 下稳定可见
   - 因为跳板阶段依赖它拿到：
     - `current_thread`
     - 当前 core 的 transition stack 信息
     - 统一内核 CR3 自身或其快速定位信息

3. `current_thread`
   - 跳板代码切换到统一内核 CR3 后，必须还能继续读到当前线程对象
   - 至少要能继续访问：
     - `current_thread` 指针本身
     - `ThreadBlock` 主体
     - `stack_levladdr`
     - 线程上下文信息

B. 切栈必需对象

4. `current_thread->stack_levladdr`
   - 这是第二步里最关键的核对项
   - 必须确认：在统一内核 CR3 下，它对应的线程真实内核栈那块内存是可访问的
   - 这里不以 `mglb(...)` 为规则
   - 这里真正要确认的是：
     - 统一内核页表是否直接覆盖这块内存
     - 切到统一内核 CR3 后能否直接把它作为切栈目标

5. 线程真实 kernel stack 顶部计算规则
   - 必须能在统一内核 CR3 下直接算出：
     - `thread_real_kstack_top`
   - 也就是：
     - 这块内存的基址可见
     - `stack_size` 可读
     - 目标 `ESP/RSP` 可安全写入

C. 切到统一内核 CR3 后继续执行必需对象

6. 内核 text / rodata / data / bss
   - 否则切过去后连后续 trampoline / C 入口都跑不了

7. GDT / IDT / TSS
   - 至少要保证切到统一内核 CR3 后，
     当前内核控制流依赖的描述符表和 TSS 仍是稳定可访问的

8. 中断控制与关键 MMIO
   - 包括 APIC / PIC / 设备寄存器区
   - 否则切过去后中断确认、时钟、IPI 等都可能失效

9. 基本内核堆 / mempool
   - 后续正常内核路径要继续工作，不能切过去后丢掉基本分配能力

10. 核心内核对象
   - 至少包括：
     - `ThreadBlock`
     - `ProcessBlock`
     - 调度器核心队列 / 当前线程链路
     - 文件系统与驱动主对象
   - 这一步不要求一次列尽全部大对象，但必须先列出入口后立刻可能访问到的最小闭包

D. 第二步的核对方法

第二步先不改入口汇编。

先按下面顺序做只读核对：

1. 现有 `_Mapping_Core_Stack()` 地址是否已被 `kernel_paging` 覆盖
2. `PERCORE` 区在统一内核 CR3 下是否稳定可见
3. `ThreadBlock` 与 `current_thread` 链路在统一内核 CR3 下是否稳定可见
4. `stack_levladdr` 在统一内核 CR3 下是否真可直接访问
5. 切到统一内核 CR3 后，能否直接把 `stack_levladdr + stack_size - delta` 作为切栈目标
6. 切到统一内核 CR3 后，后续 trampoline 所需代码页是否仍可执行

E. 第二步的通过标准

只有当下面这句话成立，第二步才算通过：

“在 per-core transition stack 上，拿到 `PERCORE->current_thread` 之后，
切到统一内核 CR3，仍然能直接算出并访问当前线程真实 kernel stack 顶部，
然后安全把 `ESP/RSP` 切过去。”

如果这句话还不能成立，就不能进入第三步改入口。

第二阶段代码核对结果（基于当前代码）

下面这部分不是目标设计，而是对“当前代码是否已经满足第二阶段前提”的核对结论。

一、已经确认成立的部分

1. per-core transition stack 在 kernel_paging 下可见

证据：

- `taskman-new.cpp` 中 `_Mapping_Core_Stack(Paging& paging)` 会把 transition stack 映射进传入页表
- `Taskman::Initialize()` 里明确调用了：
  `_Mapping_Core_Stack(kernel_paging)`

结论：

- x8632:
  `0xFFFFF000u - cpu_i * 0x1000u` 这组地址在 `kernel_paging` 下可见
- x8664:
  当前 cpu0 的
  `0x0000FFFFFFFFF000ull`
  在 `kernel_paging` 下可见

2. per-core transition stack 在用户进程页表下也可见

证据：

- `_Taskman_Create_Paging()` 在创建 ring3 进程页表时调用：
  `_Mapping_Core_Stack(ppb->paging)`

结论：

- 用户态进入内核时，CPU 先落到 transition stack 这件事，与当前页表模型是兼容的

3. `current_thread` / `ThreadBlock` 主体在统一内核 CR3 下原则上可见

证据：

- `ThreadBlock` / `ProcessBlock` 本身都是内核侧对象
- `Taskman::current_thread(cpuid)` 实际来自：
  - `PERCORE->current_thread`
  - 或全局 `PCU_CORES_current_thread`
- 这些对象都不是用户态页里的对象

结论：

- 只要统一内核 CR3 继续使用当前 `kernel_paging` 语义，
  `current_thread` 链路本身不是第二阶段的主要障碍

4. 线程真实 kernel stack 顶部的“计算信息”是齐全的

证据：

- `ThreadBlock` 持有：
  - `stack_levladdr`
  - `stack_size`
- 现有代码已经大量使用：
  `context.kernel_sp = _IMM(stack_levladdr) + stack_size - 0x10`

结论：

- “如何算出线程真实 kernel stack 顶部”不是问题
- 第二阶段真正的问题是：
  切到统一内核 CR3 以后，这个地址是否真的可访问

二、当前最关键的分歧点

第二阶段真正卡住的不是 transition stack，也不是 `current_thread`，
而是：

`current_thread->stack_levladdr` 在统一内核 CR3 下是否稳定可访问

这里 x8632 和 x8664 的结论不一样。

三、x8632 现状结论：第二阶段暂时不能直接判定通过

证据：

1. `memoman-init.cpp` 当前只给 `kernel_paging` 建了：
   - `0x00000000 -> 0x00000000` 长度 `0x10000000`
   - `0x80000000 -> 0x00000000` 长度 `0x10000000`

2. 线程真实内核栈那块内存现在来自：
   `mempool.allocate(...)`
   或 `mem.allocate(...)`
   用户当前确认：到现在还没有申请的物理内存突破 `0x10000000`

结论：

- 在 x8632 下，
  统一内核页表当前覆盖了前 `0x10000000` 物理范围
- 在“线程真实内核栈那块内存仍落在此前 256MB”这个前提下，
  切到统一内核 CR3 后可以直接把 `stack_levladdr` 这块内存作为切栈目标
- 结合当前使用现状，这一前提可接受

更准确地说：

- “transition stack 可见”已经通过
- “`current_thread` 可取到”已经通过
- “切到统一内核 CR3 后，线程真实内核栈那块内存可直接访问”
  在当前物理分配范围前提下通过

四、x8664 现状结论：第二阶段通过

证据：

1. `memoman-init.cpp` 在 x8664 下：
   - 低地址 identity map 了
     `0x00000000 -> 0x00000000`
     长度 `0x100000000ULL * 16`
   - 即低地址直接映射了 64GB

2. x8664 进程页表创建时：
   - 映射了高半区内核窗口
   - 映射了 `PERCORE_VBASE`
   - 映射了 `_Mapping_Core_Stack()`

结论：

- 只要线程真实内核栈那块内存仍位于当前低 64GB 物理范围内，
  统一内核 CR3 下访问它没有结构性障碍
- 按当前代码和常规分配规模看，这一前提是成立的

所以：

- x8664 第二阶段通过
- 后续更像是“把入口汇编改成按这个模型走”，不是“先补大块映射缺口”

五、`PERCORE` 这一项的真实结论

这项需要特别说清楚。

1. x8664 用户进程页表中，`PERCORE_VBASE` 已明确映射
2. 但当前 `kernel_paging` 代码里，没有看到把 `PERCORE_VBASE` 本身再次映射进去
3. 当前内核代码很多时候直接通过：
   - `Taskman::PCU_CORES_PERCORE[i]`
   - 或直接对象指针
   来访问 per-core 结构

结论：

- 如果未来 trampoline 设计要求“切到统一内核 CR3 后，仍通过 `PERCORE_VBASE` 固定虚拟地址访问 PERCORE”，
  那这一点当前代码还没有被完全证明
- 但如果 trampoline 只要求：
  - 在入口早期先拿到 `PERCORE*`
  - 后续继续拿这个指针用
  那现有代码路径基本是够的

所以这项目前结论是：

- `PERCORE` 对象本身可用
- `PERCORE_VBASE` 作为统一内核 CR3 下的固定访问窗，当前代码证据还不完整

六、第二阶段最终判定

x8632:

- 通过（以前 `0x10000000` 物理范围前提为条件）
- transition stack 可见
- `current_thread` 可取
- 线程真实内核栈那块内存在当前物理分配范围内可直接作为切栈目标

x8664:

- 通过
- 当前代码已经具备：
  - transition stack 可见
  - `current_thread` 可取
  - 统一内核 CR3 下大范围 identity map
  - 线程真实内核栈那块内存可直接作为切栈目标

七、因此第三步前的结论

第二阶段已经可以结束。

下一步不再纠缠 `mglb(...)` 或高半区导出地址，
而是直接进入第三步：

- 把“`stack_levladdr` 是线程真实内核栈那块内存，
  trampoline 切到统一内核 CR3 后直接切到这块内存顶部”
  写成正式入口规则

第三步：确定入口 trampoline 的精确时序，并与现状逐项对比

这一节的目标不是马上改汇编，
而是先把“现在入口到底做了什么”和“目标入口应该做什么”并排写清楚。

一、x8632 当前 syscall 入口实际时序

文件：

- `prehost/atx-x86-flap32/atx-x86.asm`
- `unisym/lib/asm/x86/interrupt/ruptable.asm`

当前 `Handint_SYSCALL_Entry` / `Handint_INTCALL_Entry` 的实际顺序是：

1. CPU 已按当前 TSS.ESP0 落到 ring0 栈
2. `PUSHFD`
3. `CLI`
4. `PUSHAD`
5. `CALL PG_PUSH`
6. `PG_PUSH` 内部：
   - 保存原来的 `DS/SS`
   - 切段寄存器到 `SegData`
   - 读取当前 `CR3`
   - 记录旧 `ESP`
   - 直接把 `CR3` 切到 `ROOT_PAGING`
   - 用 `ConvertStackPointer` 把“旧栈上的线性地址”换算成 `ROOT_PAGING` 下可访问的地址
   - 继续沿用这一份已经压好的现场
7. 回到入口桩后，取 `CallgateFrame*`
8. 直接调用 `Handint_SYSCALL`
9. 返回后 `PG_POP`
10. `POPAD`
11. `POPFD`
12. `RETF` / `IRETD`

当前这一套的关键特征：

- 没有 per-core transition stack -> 线程真实内核栈 的二段式切栈
- 是“先在当前 ring0 栈上压好大部分现场，再由 `PG_PUSH` 把这份栈地址换到 root paging 下继续用”
- 也就是说：
  当前 syscall 入口真正依赖的是“同一份现成栈内容跨 CR3 继续使用”
  而不是“尽快切到当前线程真实内核栈，再在那里建立正式上下文”

二、x8632 当前普通中断入口实际时序

文件：

- `prehost/atx-x86-flap32/atx-x86.asm`

`IRQ_TRAMPOLINE -> Handint_Common_Stub` 的实际顺序是：

1. 先压入 dummy error code 和 interrupt id
2. `PUSHAD`
3. `CALL PG_PUSH`
4. 把当前 `ESP` 当作 `HardwareInterruptFrame*` 传给 `interrupt_dispatcher`
5. `CALL interrupt_dispatcher`
6. `CALL PG_POP`
7. `POPAD`
8. `ADD ESP, 8`
9. `IRETD`

当前这一套的关键特征：

- 普通中断入口和 syscall 入口一样，
  也是先在“当前 ring0 入口栈”上形成现场，再靠 `PG_PUSH` 切到 root paging 继续沿用
- 仍然没有“先落跳板栈，再切线程真实内核栈，再在那里建立正式 trap frame”这一步

三、x8664 当前 syscall 入口实际时序

文件：

- `prehost/atx-x64-uefi64/atx-x64.asm`

当前 `Handint_SYSCALL_Entry` 的实际顺序是：

1. `SWAPGS`
2. `MOV [GS:136], RSP`
   - 保存用户态 RSP 到 `PERCORE.scratch`
3. `MOV RSP, [GS:128]`
   - 直接切到 `PERCORE.kernel_rsp`
4. 在这根内核栈上手工 `PUSH` 出一份 syscall frame
5. 保存一组易失寄存器
6. `CALL PG_PUSH`
7. `CALL Handint_SYSCALL`
8. `CALL PG_POP`
9. 从同一根栈上恢复上下文
10. `SWAPGS`
11. `SYSRET`

当前这一套的关键特征：

- x8664 已经显式经由 `PERCORE.kernel_rsp` 进内核
- 但进入后仍然是“在这根现成内核栈上直接构造完整 frame”
- 没有再细分成：
  - per-core transition stack
  - 线程真实内核栈
  这两个阶段

四、x8664 当前普通中断入口实际时序

文件：

- `prehost/atx-x64-uefi64/atx-x64.asm`

`Handint_Common_Stub_64` 的实际顺序是：

1. 若来自 Ring 3，则 `SWAPGS`
2. `PUSHA64`
3. `CALL PG_PUSH`
4. 把当前 `RSP` 作为 `HardwareInterruptFrame*` 传给 `interrupt_dispatcher`
5. `CALL interrupt_dispatcher`
6. `CALL PG_POP`
7. `POPA64`
8. 若返回 Ring 3，则 `SWAPGS`
9. `ADD RSP, 16`
10. `IRETQ`

当前这一套的关键特征：

- 仍然是“先在当前进入内核后的那根栈上建完整现场，再继续跑”
- 没有切到“线程真实内核栈那块内存对应的恒等映射地址”

五、目标 trampoline 时序

无论 x8632 还是 x8664，目标都应统一成两段式：

阶段 A：per-core transition stack 阶段

1. CPU 从用户态进入 ring0
2. 自动落到固定 `TSS.ESP0/RSP0`
3. 在 transition stack 上只保存最少量早期现场
4. 读取 `PERCORE`
5. 找到 `current_thread`
6. 切到统一内核 CR3
7. 计算“线程真实内核栈那块内存”在统一内核页表主体恒等映射下的顶部地址
8. 把 `ESP/RSP` 切到那里

阶段 B：线程真实内核栈阶段

1. 把阶段 A 留下的必要早期现场转移到线程真实内核栈
2. 在这根线程栈上建立完整 trap frame / syscall frame
3. 再进入 C 级分发逻辑
4. 只有到这里，才允许进入可调度状态

六、现状与目标的核心差异

当前实现：

1. 先在当前 ring0 入口栈上压完整或接近完整的现场
2. 通过 `PG_PUSH` 把当前现场对应的地址换到 root paging 下继续用
3. 不区分“transition stack 阶段”和“线程真实内核栈阶段”

目标实现：

1. 先只在 per-core transition stack 上停留极短时间
2. 尽快切到统一内核 CR3
3. 再尽快切到线程真实内核栈
4. 完整上下文只在线程真实内核栈上建立和长期保存

七、第三步真正要改动的不是哪一行，而是哪条原则

当前入口路径默认原则是：

- “先建现场，再换页表后继续沿用这根栈”

第三步要把它改成：

- “先最小落地，再切统一内核页表，再切线程真实内核栈，最后才建完整现场”

这就是第三步最核心的对比结论。

阶段 3：线程真实内核栈那块内存接入统一内核页表

1. 线程创建时把 `stack_levladdr` 注册进统一内核页表
2. 线程销毁时解除映射
3. 验证从统一内核页表下能访问每个线程栈

阶段 4：修改 syscall / interrupt 入口跳板

1. 用户态进内核先落 per-core 小栈
2. 跳板中：
   - 取 `current_thread`
   - 切统一内核 CR3
   - 切到 `current_thread->stack_levladdr` 顶部
3. 再跳转常规内核入口

阶段 5：从调度热路径移除 rebind

1. `BindCurrentKernelEntryStack()` 不再 remap stack window
2. `Schedule()/SleepAndRelease()` 不再调用 `RebindThreadKernelStackWindow()`
3. 只做线程切换本身

阶段 6：清理旧 fixed window 运行栈依赖

1. 检查所有默认假设“当前运行栈在 GetCoreRingStackBase()”的代码
2. 改成适应“跳板栈只做入口、运行栈在线程真实内核栈那块内存上”


十四、最危险的坑

1. 统一内核页表缺映射
   - 一旦切过去后，某些内核对象看不见，立刻死

2. 用户指针还按旧习惯直接访问
   - 在统一内核页表下会直接 page fault

3. 中断嵌套时又回到跳板栈处理不当
   - 可能破坏当前线程执行栈

4. `current_thread` 与真正运行线程不一致
   - 跳板切错栈会直接炸

5. 线程销毁后真实内核栈那块内存释放过早
   - 统一内核页表还保留旧映射或仍有 core 在用


十五、需要重点修改的文件

1. `mecocoa/schedul.cpp`
   - 重新定义 `BindCurrentKernelEntryStack`
   - 去掉调度热路径中的 `RebindThreadKernelStackWindow`

2. `mecocoa/taskman-new.cpp`
   - 每 core 跳板栈初始化
   - 统一内核页表准备

3. `mecocoa/taskman.cpp`
   - 线程真实内核栈那块内存注册/注销到统一内核页表

4. x86 入口汇编
   - syscall / interrupt trampoline
   - 从 per-core 小栈切到线程真实 kernel stack

5. 用户指针访问相关路径
   - syscall
   - signals
   - 文件读写桥接
   - exec/fork 栈构造


十六、和当前 Freetype 慢启动问题的关系

这套方案如果落地成功，最大的收益是：

1. `Task_FileSys <-> Task_Hdd_Serv` 高频阻塞/唤醒
2. 不再叠加“每次切线程重绑整段 stack window”

也就是说，它不会减少 FAT 的碎读次数，
也不会直接减少 HDD 代理读的 sector 数量，
但它会显著降低“每次代理切换”的调度附加成本。

所以它解决的是：

- “为什么最近同样的碎读模式被放大得特别厉害”

而不是：

- “为什么 FAT 自身对碎读不友好”


十七、一句话总结

本方案的本质是：

把 `ESP0/RSP0` 退化为“每核固定入口小栈”，
把线程真正执行期的内核栈放回“每线程自己的真实内核栈那块内存”，
再通过“统一内核页表 + 入口跳板切栈”避免：

1. 每次调度 remap fixed window
2. 每线程长期占 permanent kernel stack VA

这是比“per-thread permanent kernel VA”更适合大线程规模的一条路线。


十八、入口栈与跳板栈复用

当前 `_Mapping_Core_Stack()` 已经在为每个 core 映射一小段高地址栈。

这段栈可以复用，不必强行拆成两种不同资源。

更准确的命名建议是：

- per-core transition stack

它同时承担两种短暂角色：

1. 用户态 -> 内核态的入口跳板栈
2. 内核态 -> 用户态的返回整理栈

也就是说：

1. 进入内核时
   - CPU 自动使用 `TSS.ESP0/RSP0`
   - 落到这根 per-core transition stack

2. 返回用户态时
   - 从线程真实 kernel stack 切回这根 per-core transition stack
   - 摆好 `iret/iretq` 所需现场
   - 再返回用户态

这样做的原因：

1. 入口阶段本来就是短暂停留
2. 返回阶段本来也是短暂停留
3. 二者都不应承载线程长期执行
4. 所以复用一根 per-core 小栈是自然的

但要强调两条规则：

1. 这根栈只做“过渡”
   - 不承担线程正常内核执行栈
   - 不在上面跑复杂逻辑

2. 它必须在：
   - 用户进程当前 CR3 下可见（用于入口）
   - 统一内核 CR3 下也可见（用于切页表后继续过渡、以及返回前整理）

关于大小：

第一阶段先按当前 `_Mapping_Core_Stack()` 已有大小工作。

也就是说：

1. 不因为重定义语义就先扩新的 transition stack VA
2. 先证明“现有映射页 + 尽快切到线程真实 kernel stack”这条路径成立
3. 是否扩页，留到后续单独评估

所以这根栈未来的语义应当明确成：

- per-core transition stack
- 入口和返回复用
- 不作为线程主执行栈


十九、利用 PERCORE，短暂使用跳板栈，直接切换 ESP 到线程真实内核栈

这是本方案里最有价值的一条实现技巧。

目标是：

1. 在 per-core transition stack 上只停留极短时间
2. 不在 transition stack 上保存大块上下文
3. 尽快切到当前线程自己的真实 kernel stack
4. 在真实 kernel stack 上保存完整 trap/syscall 上下文

推荐入口模型如下。

阶段 A：极早期 trampoline（运行在 per-core transition stack 上）

只做最少动作：

1. 关中断
2. 取得当前 CPU 对应的 `PERCORE*`
3. 从 `PERCORE->current_thread` 取得当前线程
4. 切换到统一内核 CR3
5. 计算当前线程真实 kernel stack 顶部
6. 直接把 `ESP/RSP` 切到线程真实 kernel stack

这一步的关键点是：

- 不要在 transition stack 上保存太多寄存器
- 尽量只使用极少量 scratch 寄存器
- 复杂保存留到切到线程栈之后再做

阶段 B：真实上下文保存（已经在线程真实 kernel stack 上）

在这一步再去做：

1. 保存通用寄存器
2. 构造 trap frame / syscall frame
3. 保存错误码、用户态返回地址、用户态 `SS/ESP`、`CS/EIP`、`EFLAGS`
4. 然后才进入 C 级内核入口

这里要特别强调：

- 如果阶段 A 已经在 transition stack 上形成了任何后续恢复所需的早期现场，
  那么在进入阶段 B 后，必须先把这些必要信息整理并转移到线程真实 kernel stack 上
- 不能让“恢复当前线程所需的信息”长期留在 per-core transition stack 上

为什么这样更好：

1. transition stack 只承担“接住 + 转栈”
2. 线程自己的可调度上下文一开始就落在自己的真实 kernel stack 上
3. 后续即使阻塞、抢占、调度到别的线程，也不会再依赖 per-core transition stack 上的旧现场

这条规则必须写死：

任何可能跨调度边界存活的内核上下文，
都不能长期留在 per-core transition stack 上。

换句话说：

在允许调度之前，必须完成两件事：

1. 切到线程真实 kernel stack
2. 把后续恢复所需的完整上下文保存在该线程自己的 kernel stack 上

否则如果中途调度：

1. 当前线程的恢复信息仍留在 per-core transition stack
2. 同 core 上别的线程进内核又会复用这根栈
3. 就会造成上下文覆盖

因此，这里要建立一个明确的内核规则：

- transition stack 阶段：must_not_schedule
- 只有在 trap frame 已迁移/建立到线程真实 kernel stack 后，线程才进入正常可调度状态


二十、对 PERCORE 的要求

为了支持“极短 trampoline -> 直接切线程栈”，`PERCORE` 应承担最早期定位职责。

最少需要稳定拿到：

1. `current_thread`
2. 当前 core 的 transition stack top
3. 统一内核 CR3（建议也放进 PERCORE 或能快速全局取得）

可选扩展：

1. `current_trapframe`
2. 当前 core 是否正处于 transition 阶段
3. 返回用户态时使用的 transition frame 顶指针

也就是说，PERCORE 在这套设计里最大的价值不是保存大对象，
而是让入口汇编能用最少指令找到：

- 我是谁（哪个 core）
- 当前线程是谁
- 我要切到哪根真实 kernel stack
- 我要切到哪个内核 CR3


二十一、这部分设计的边界

“利用 PERCORE 短暂使用跳板栈，直接切到线程真实 kernel stack，再在那边保存上下文”
是推荐主路径，但要明确边界：

1. 它适用于普通 syscall / trap / 大多数可控中断
2. 对 NMI / 双故障 / 崩溃恢复路径，要单独设计更保守的专用栈策略

也就是说：

- 普通路径可以复用 per-core transition stack
- 极端异常路径不应盲目套这条主路径


二十二、对整体方案的修正性总结

基于上面的讨论，本方案现在应明确为：

1. 每 core 一根固定的 transition stack
   - 入口/返回复用

2. 每线程一块真实内核栈内存

3. 用户态进入内核：
   - 先落到 transition stack
   - 借助 PERCORE 快速定位 current_thread
   - 切统一内核 CR3
   - 直接切到线程真实 kernel stack
   - 然后才在真实 kernel stack 上保存完整上下文

4. 返回用户态：
   - 从线程真实 kernel stack 切回 transition stack
   - 整理 `iret/iretq` 现场
   - 返回用户态

5. 调度器不再承担 stack remap 工作
   - 线程切换只切上下文
   - 不再改 fixed window 映射
