
我现在想把原来的单线程任务的内核转换为多线程的任务内核
- 我把单线程任务的内核的最新跑通的成果放在mecocoa的最后一次commit中
- commit之后我设计了一下 Spinlock和Mutex，并初步策划了 ThreadBlock
- 需要更改原本的调度机制，由进程转变为线程
- 通信机制(message)需要由原本的进程转变为线程间
- 现在进程回收之后进程号没有得到复用，看看是什么原因
- 各种系统调用(syscall)也需要适配

---

我现在想把原有的 Mecocoa 单线程任务内核架构，重构并过渡为真正的多线程任务内核。

我把单线程内核最新跑通的基础代码，放在了当前工作区的最后一次 commit 中。

我已经设计并实现了用于内核同步的 Spinlock（自旋锁）和 Mutex（互斥锁），并初步规划了 ThreadBlock (TCB) 的数据结构。

请协助我完成以下底层逻辑的拆分与重构：

数据结构解耦： 将原有的 ProcessBlock 严格拆分为资源管理者 (PCB) 和 执行调度单元 (TCB)。PCB 负责持有 paging (页表)、heaptop/heapbtm (堆区) 和 pfiles (文件描述符表)；ThreadBlock 负责持有 context (CPU 上下文)、stack_lineaddr (独立栈) 和调度状态。

调度器改造： 调度队列和时间片轮转的最小单位由进程变更为线程。请特别注意在上下文切换 (SwitchTaskContext) 时进行同进程判定：如果是同一个进程内的线程切换，必须跳过 CR3 (或 satp) 页表的切换，以避免 TLB Flush 性能损耗。

锁机制的落地： 在多线程环境下，请使用我新设计的 Mutex 或 Spinlock 来保护 PCB 内部的共享资源（例如多个线程并发操作文件描述符表或申请堆内存时的互斥）。

通信机制 (IPC) 升级： 现有的 Message 机制需要从“进程到进程”扩展为支持“线程到线程”以及“进程到线程”的路由分配。

Syscall 生命周期适配： 请区分进程级与线程级的系统调用。调整现有的 exit 与 fork 逻辑，并引入专门用于创建新线程（为其分配独立用户栈/内核栈）和销毁单个线程的机制。

修复 PID 泄漏 BUG： 目前进程回收后，PID 并没有得到复用。请检查现有的 chain 链表和 min_available_pid 相关的分配逻辑，修复 PID 释放与回收复用的机制。

（目前没有创建线程、等待线程的系统调用，现在先不考虑实现，只要保证之前有的功能要兼容即可）
（需求：中文回答 英文注释 TAB缩进 尽量少加入新的注释，避免注释泛滥，尽量见到代码字面就知道意思 原来存在的注释保留 不要去掉 ）

---

现在用户态程序无法正常加载。或者正常加载后无法正常进行系统调用，请找明原因。

（COMMIT 20260419+）

---

（需求：中文回答 英文注释 TAB缩进 尽量少加入新的注释，避免注释泛滥，尽量见到代码字面就知道意思 原来存在的注释保留 不要去掉 ）【构建让我来，不要擅自make run。】

现在的多线程适配存在局限性，请分析并解决以下问题：

1、代码中普遍使用 CurrentPID 而不是 CurrentTID，尤其是syscall，只考虑 main_thread；
2、现在pcurrent记录的是pid，但是应该记录是当前线程的thread_id。
3、msg_send现在仍然是ProcessBlock为单位（应为ThreadBlock）。IPC 通信机制尚未真正支持多线程，在 msg_send 和 msg_recv 中，仍然硬编码了绑定 main_thread。
4、TID 分配的 SMP 竞态与性能灾难：在 AppendThread 中：LocateThread 需要遍历全系统的进程和线程链表，这里是一个 $O(N^2)$ 的操作。可以参考Taskman::chain再加一个 Taskman::thchain

---

修改之后 用户态程序无法正常调度，或者正常调度了在系统调用里卡死了。

新的代码中调用了 operator delete(void* ptr, stduint size)，而且释放了不该释放的内存切片。

---

（需求：中文回答 英文注释 TAB缩进 尽量少加入新的注释，避免注释泛滥，尽量见到代码字面就知道意思 原来存在的注释保留 不要去掉 ）【构建让我来，不要擅自make run。】

现在的多线程适配存在局限性，请分析并解决以下问题：

一、`pcurrent` 数组现在完全是冗余的，且极易与 `current_thread` 发生状态脱节（Desync）。

**修复：** 彻底删掉 `pcurrent`。既然我们已经有了极其可靠的 `current_thread` 数组，所有 ID 查询都应该直接透传给它。

```
static stduint CurrentTID() { return current_thread[getID()]->tid; }
static stduint CurrentPID() { return current_thread[getID()]->parent_process->pid; }
```

注意：在 `Schedule` 中，删除对 `CurrentPID()` 和 `CurrentTID()` 的赋值，只更新 `current_thread[cpuid] = new_tb;` 即可。

二、TID 分配的 SMP 竞态与 $O(N^2)$ 性能灾难依然存在

在 `Append` 和 `AppendThread` 中通过同时遍历 `chain` 和 `thchain` 来避免 PID 和 TID 冲突：

```C++
while (true) {
    bool conflict = false;
    for (auto nod = chain.Root(); nod; nod = nod->next) { ... }
    for (auto nod = thchain.Root(); nod; nod = nod->next) { ... }
    if (!conflict) break;
    target_id++;
}
```

**问题：** 
1. **性能依然是 $O(N^2)$：** 你不仅在遍历，而且嵌套遍历了两个链表，随着任务增多，创建线程的速度会呈指数级下降。
2. **SMP 竞态依然致命：** 没有任何锁保护。如果 CPU 0 和 CPU 1 同时进入这个 `while` 循环，它们扫出来的链表是一样的，最终会给两个不同的线程分配完全相同的 `target_id`，导致内核链表彻底崩溃。
**修复：** 抛弃循环遍历分配的思路，直接使用硬件级别的**全局原子自增计数器**（这也是现代操作系统的做法）。

```C++
// 在 Taskman.cpp 顶部定义
static stduint next_global_id = 1;

// 在 AppendThread 中直接替换为：
bool Taskman::AppendThread(ThreadBlock* task) {
    if (task->parent_process && task->parent_process->main_thread == task) {
        task->tid = task->parent_process->pid;
    } else {
        // O(1) 且多核绝对安全
        task->tid = __atomic_fetch_add(&next_global_id, 1, __ATOMIC_SEQ_CST);
    }
    
    // ... 插入 thchain 和就绪队列的代码保持不变 ...
}
```

三、 IPC (Message) 机制的潜在空指针危险

在 `msg_recv` 中：

```C++
else if (foo != INTRUPT) {
    fo_th = Taskman::LocateThread(foo);
    if (!fo_th) {
        if (auto fo_pb = Taskman::Locate(foo)) fo_th = fo_pb->main_thread;
    }
    if (!fo_th) return 1; // 拦截了空指针
    
    // 但是这里：
    if (fo_th->block_reason == ThreadBlock::BlockReason::BR_SendMsg && ...)
```

这段逻辑现在是安全的（因为拦截了 `!fo_th`）。但在下方的 `block self to wait for msg` 分支中：

```C++
} else {
    ThreadBlock* fo_th_tgt = Taskman::LocateThread(foo);
    if (!fo_th_tgt) {
        if (auto fo_pb = Taskman::Locate(foo)) fo_th_tgt = fo_pb->main_thread;
    }
    to_th->recv_fo_whom = fo_th_tgt; // 如果 foo 是一个不存在的 PID/TID，这里会存入 nullptr
}
```

**隐患：** 如果用户态传入了一个无效的 `foo`，`fo_th_tgt` 会是 `nullptr`。将 `nullptr` 赋给 `recv_fo_whom`，在你的设计中等同于 `ANYPROC`（接收任意消息），这违背了调用者的意图，会导致严重的安全越权。 **修复：** 如果找不到目标线程/进程，应直接返回错误码（例如 `return -1;`），拒绝阻塞当前线程。

（COMMIT 20260420）

---

（需求：中文回答 英文注释 TAB缩进 ）【构建让我来，不要擅自make run。】
现在的多线程适配存在局限性，请分析并解决以下问题：

调度器的 Idle 逻辑错误（会导致系统死锁或跑飞）：在 Taskman::Schedule 中：
auto new_tb = PickNext();
if (!new_tb) new_tb = current_thread[cpuid]; // At least run self or idle thread if nothing else
问题： 假设 current_thread 是因为调用了 Mutex::Acquire 失败而被挂起（此时 old_tb->state = TBS::Pended）。如果此时就绪队列为空，PickNext() 返回 nullptr，你的代码会强行把 new_tb 设置为当前这个处于 Pended 状态的线程，并继续往下执行！被阻塞的线程会被强行拉回 CPU，破坏了互斥锁的语义。绝对不能把 Pended 状态的线程拉回 CPU。当 PickNext() 为空时，必须切换到一个专门的 Idle 线程（这个线程在初始化时创建，内部是一个死循环执行 hlt / pause 指令）。如果暂时不想实现单独的 Idle 线程，至少要原地自旋等待中断，而不是 return 给调用者。
如果要实现Idle进程：用同一段 Taskman::Idle() 作为 idle即可

---

# 20260422 Lock 与 调度

```
【20260421 给调度机制加锁】

（需求：中文回答 英文注释 TAB缩进 
尽量见到代码字面就知道意思，可以适当加注释说明 
原来存在的注释保留 不要去掉 ）

假设CPU0现在从线程a且b过程，CPU1刚好发现a就绪。

我的方案：我创建一个数组 Taskman::switching_out_threads[PCU_CORES_MAX] ，存放每个CPU正要切出的thread标号。同时每个线程块（ThreadBlock）引入just_schedule，每次schedule和解锁前置位。这样每次调度之前都判断一遍。当CPU0下一次切换另一个线程时，a不就从黑名单中出来了。CPU1检索到a时，先去a的控制块看看just_schedule字段有没有设置，看到A被占用之后直接找别的线程去调度，不会傻等，如果A的just_schedule为0，则表示没有问题，在判断索引0还是a就直接把数组索引0置0即可（必须判断，不然容易错误覆写）。除此之外，CPU0发生时钟中断触发schedule时，看到a在数组索引0号位中，此时任务切换一定结束了，不管时间片如何都可以把索引0的指针置零，表示线程a被释放了。

just_schedule由汇编操作在把a的上下文都保存完之后再置0。

注意加volatile 关键字，确保 switching_out_threads 数组在内存中是自然对齐的（4字节或8字节），这样对指针的读写操作在硬件层面上就是原子的，不会出现“写了一半地址”的情况。

如果队列里除了 A，没有别的线程了呢？这说明当前线程确实比较少了，让CPU0单核执行好了，CPU1 idle 休息一下。这是没办法避免的事。

这样可以实现在Taskman::Schedule的 SwitchTaskContext 提前开锁，并且可以彻底删除 “task_switch_enable” 这个变量

你需要做的事情
- 实现上文提到的 just_schedule 逻辑，当然还要更新 SwitchTaskContext，各个架构的相关汇编文件如下
	- D:\her\unisym\lib\asm\x86\task.asm
	- D:\her\unisym\lib\asm\x64\task.asm
	- RISCV32&64: D:\her\mecocoa\prehost\qemuvirt-r32\qemuvirt-rv.startup.S
- 检查 D:\her\mecocoa\mecocoa\schedul.cpp
- 其他 D:\her\mecocoa\mecocoa\*.cpp 的相关逻辑
- 还需要关注 Taskman::SleepAndRelease(Spinlock* lk)

```

# 20260530 正式多线程

```
为什么要修改CR3？是因为x86在 call gate 的时候，从 R3-R0 会换页表，如果不记录，那么再次调度回来的时候用旧页表返回 R0 代码必然出错（现在的设计是，R3必须由R0来返回）

现在我们应该检查
1、在R3-R0过程中，用户栈是否必须物理连续
2、在R3-R0过程中，内核栈是否必须物理连续

如果，用户栈不是必须物理连续，那皆大欢喜，我们可以直接转换E/RSP到物理地址用，进而只要求内核解决把用户地址空间的栈过继给新线程即可
```

R3→R0 切换时用的是内核栈

x86 调用门特权切换时，CPU 硬件做的是：

从 TSS.SS0:ESP0 加载 → 切换到内核栈

把用户的 SS, ESP, EFLAGS, CS, EIP push 到内核栈上

TSS.ESP0 指向的就是 `_NORMAL_RINGSTACK` top，这是内核栈，**用户栈完全没有被访问**，只有其 ESP 值被作为数据保存。

```
x64 用的是 syscall 指令，这种情况呢？请检查一下用户栈
```

x64 `syscall` 时用户栈的情况

`syscall` 指令本身的行为（x86-64 硬件规范）：
- 保存 RIP → RCX，保存 RFLAGS → R11
- **不切换 RSP**（不像 Call Gate），**不切换 CR3**，**不访问 TSS**
- 直接跳到 LSTAR 指向的 `Handint_SYSCALL_Entry`，**此时 RSP 还是用户栈的虚拟地址**

然后看 `Handint_SYSCALL_Entry`（第 149-150 行）
**整个寄存器保存（第 150-165 行的 PUSH 序列）都发生在 `PG_PUSH` 之前**，也就是说，这些 PUSH 操作全部压在**用户栈的虚拟地址上**，使用的还是**用户进程的 CR3**（`syscall` 没有切 CR3）。

`PG_PUSH` 被调用时，它拿到的 RSP 就是这段 PUSH 后的用户栈指针，然后走 `Convert_VA_to_PA` 翻译成物理地址，再切换 CR3。

结论：用户栈的约束

在 `PG_PUSH` 调用之前，代码压了约 **15 个 QWORD（120 字节）** 到用户栈上。`Convert_VA_to_PA` 是**按页翻译单一地址**，它翻译的是那一刻的 RSP 值对应的物理页，**不管跨页问题**。

因此：

**用户栈必须满足：`PG_PUSH` 时的 RSP 所在的 4KB 物理页，在内核恒等映射下必须可访问**。这一点由 `mempool` 恒等映射保证——用户栈物理页本身就在恒等映射区，所以**事实上满足**。

但若用户栈某一帧恰好跨了 4KB 页边界，`PG_PUSH` 只翻译了低地址那页，高地址那页在内核页表里**不一定有映射**。

```
正好说反了。我们研究的问题是什么？不是现在用户栈是用户malloc出来的吗？malloc的地址肯定物理不连续啊
```


内核和用户页表都有的映射（0xFFFF_xxxx 内核正则空间）：
  0xFFFF_FFFF_C000_1000        _NORMAL_RINGSTACK 起始
  0xFFFF_FFFF_C000_1000
  ~ 0xFFFF_FFFF_C000_9000      NORMAL_RINGSTACK (HIGHER_STACK_SIZE=0x8000)
恒等映射区  
  0xFFFF_FFFF_FFFF_F000        higher_stacks[0] (cpu0 的跳板栈，1 page)

| |需要新分配|SMP 安全|性能|复杂度|
|---|---|---|---|---|
|TSS scratch + SWAPGS|否（复用 TSS）|✅|好|低|
|INT n|否（复用 TSS.RSP0）|✅|稍差|最低|
|SYSENTER|否（MSR）|✅|好|低但有限制|

```
我有一个想法：把 0xFFFF_FFFF_B000_0000 ~ 0xFFFF_FFFF_BFFF_FFFF 作为x64系统的 PERCPU 的映射区。（PERCPU[0]的起始地址应该 4k 对齐），成为 内核和用户页表都有的映射
```

| 方案                                        | 核心思路                                                              | 接近谁          | 优点                    | 缺点                                      | 适合现在吗        |
| ----------------------------------------- | ----------------------------------------------------------------- | ------------ | --------------------- | --------------------------------------- | ------------ |
| A. 继续修单份 ThreadBlock::context             | 仍只保留一份 context，但在 syscall/IRQ/调度里小心避免互相覆盖                         | 现状修补         | 改动最小，短期见效快            | 很脆，后面 signal/join/futex/嵌套异常还容易再炸       | 适合短期救火，不适合长期 |
| B. user_context + kernel_context 双上下文     | 用户恢复现场和内核调度现场彻底分开保存                                               | 你现在最容易落地的重构  | 语义清楚，最直接解决“被覆盖”问题     | 要改 SwitchTaskContext、syscall、IRQ、线程恢复路径 | 很适合现在        |
| C. Linux 风格“每线程内核栈 + 栈上 trap frame 为真相”   | 用户 trap frame 主要留在线程自己的 kernel stack，切换靠 kernel stack + 少量寄存器恢复   | Linux        | 体系最成熟，signal/抢占/阻塞天然顺 | 改动最大，要重塑 x86/x64 trap/return 设计         | 长期最好，短期成本高   |
| D. RISC-V 风格“单一权威 context + 独立 kernel_sp” | trap/schedule 都直接围绕线程 context 工作，context 里明确带 IP/SP/.../kernel_sp | 你仓库里的 RISC-V | 结构统一、可读性好，比现状稳定很多     | x86/x64 要重写入口保存逻辑，不能只补几行                | 很值得参考        |
| E. 用户线程仅协作式，不在 syscall/IRQ 中抢占切换          | 线程只在显式 yield/阻塞点切换，先回避复杂抢占                                        | 简化模型         | 最容易先跑通 pthread 基础功能   | 不是完整多线程抢占模型                             | 适合阶段性过渡      |

```
不是 我的理解不是这样的
2. 时钟中断：U -> Kernel(保存U上下文) -> Sch -> ... -> Sch返回(恢复Kernel上下文) -> U
你说这两份东西最后都往 ThreadBlock::context 里塞
我的意思是，U->Kernel U的上下文保存在内核栈，而不是 content ，content 没有被使用，专门用于 Schedule
Kernel->Sch 时保存内核态上下文，等调度回来的时候正好恢复

其他情况类似
```

对，你这个澄清很重要。
如果按你原本的设计，ThreadBlock::context 确实只该保存“内核态调度上下文”，而用户态现场留在每线程自己的内核栈上。

这个模型本身是说得通的，而且我觉得你原来的思路就是这个。

所以我修正一下：

对一个已经跑起来、已经进过内核的用户线程来说
你的理解是对的：
U -> Kernel：用户态现场压在该线程自己的 kernel stack 上
Schedule：只把“当前内核执行点”保存到 context
调回来：先恢复内核态执行点
再顺着这条内核栈上的 frame 链返回到用户态
这条链本身没有逻辑问题。

那现在真正的问题在哪？

我现在更倾向于说：

问题不在“普通被打断过的用户线程”，而在“新创建的用户线程第一次启动”这件事。

因为新线程和老线程不一样：

老线程：已经有过一次 U -> Kernel
它自己的 kernel stack 上确实有一套可返回的现场链
新线程：从来没进过内核返回链
它没有那份“栈上的 U->Kernel 历史现场”
所以如果你继续沿用原模型：

context 只表示“内核态调度点”
用户态恢复依赖“kernel stack 上已有的 trap/syscall frame”
那新线程第一次运行时就会缺东西：

它没有一份现成的 kernel stack frame，能让调度器恢复后再一路 iret/sysret 到用户态入口
这正好解释你现在的现象：

TNEW_CTX 打出来的 IP/SP/CS/SS 都很正常
但一真正切到新线程，就变成：
x64 CS=0x10, RIP=0x5F8F48
x86 CS=0x20, EIP=0x4C
这很像：

调度器恢复的是一份“应该继续内核执行”的上下文，但新线程根本没有一条真实的内核返回链可接。

所以问题不是你刚才那条正常时序错了，而是：

你的设计对“已经存在的用户线程”成立
但对“新建线程第一次启动”还缺一个启动桥
也就是新线程第一次运行时，需要二选一：

做一个内核 trampoline
让它看起来像“这个线程已经在内核里，马上要返回用户态”
也就是在它自己的 kernel stack 上伪造一份首次返回链
给 SwitchTaskContext 一个 first-run 特判
如果这是一个从未运行过的用户线程
不按“恢复内核 continuation”处理
而是直接按 IP/SP/CS/SS 把它送进用户态
我现在更相信：

你当前炸点主要是第 1 次启动路径没建好，不是你原本的“栈上保存用户态现场”思想错了。

所以这也解释了为什么你会觉得：

“旧模型里 context 本来就不是存 U-context 的”
这句话我现在同意
问题在于：

新线程第一次启动时，它还没有那份栈上的 U-context 历史链
但我们却试图用“恢复普通线程”的方式去启动它
这才是关键矛盾。

```
对！我想起来了！现在的关键是新线程的上下文对吧。我记得我在研究新进程上下文遇到过这个问题，并且分类讨论过，请看看我是怎么解决这个问题的
```

所以现在其实有两个层次：

|层次|结论|
|---|---|
|新线程第一次启动|你以前的解法是：context 直接作为用户态入口真相，SwitchTaskContext 伪造 IRET 帧进入 Ring3|
|线程已经跑起来后再 syscall/中断调度|现在仍然有“context 被内核态调度现场覆盖”的问题|

RISCV 的实现有明确的首次判断 （因为 mret 不知道要回哪个 ring），而 x86/x64 没有 （因为 SYSRET 靠 CS 自己就知道回 ring 3）。

```
我的想法是就用内核栈保存上下文。有一个用户态保存一次，嵌套进入就继续保存一层，应该和Linux挺像的，内核栈统一保存上下文，context 统一保存调度上下文
```

x64 syscall / interrupt rework notes
===================================

Goal
----
Reintroduce x64 multithreading support in small verified stages, instead of
replaying the previous large batch of changes from backup.

Stage 1: Restore the PERCORE abstraction
----------------------------------------
- Unified x86/x64 around a common PERCORE block.
- x86 mainly uses PERCORE.tss.
- x64 extends PERCORE with:
  - kernel_rsp
  - scratch
  - current_thread / idle_thread / switching_out_thread
- Added CurrentTB() / CurrentPB() helpers to reduce scattered direct array use.

Result:
- The kernel compiled and booted again on a clean baseline.

Stage 2: Make x64 SYSCALL use GS + PERCORE
------------------------------------------
Problem:
- x64 SYSCALL does not switch to a kernel stack automatically.
- Using the current RSP directly is unsafe for user threads with disjoint
  kernel stacks.

Solution:
- Use SWAPGS on entry.
- Store user RSP into PERCORE.scratch.
- Load the current thread's kernel entry stack from PERCORE.kernel_rsp.
- Build the synthetic syscall frame on that stack.
- Restore user RSP and SWAPGS again before SYSRET.

Required support:
- Map PERCORE_VBASE into user process page tables, because the first few
  SYSCALL-entry instructions still run under the user CR3.
- Update PERCORE.current_thread and PERCORE.kernel_rsp during scheduling.

Observed failure while bringing this up:
- Page fault at address 0x88
  Meaning:
  - GS base was effectively 0 at SYSCALL entry.
  Fix:
  - Pair SWAPGS correctly on all Ring3 return paths.

Observed failure after that:
- Page fault at address 0xFFFFFFFFB0000088
  Meaning:
  - GS base was now correct, but PERCORE_VBASE was not mapped in the current
    user page table.
  Fix:
  - Map each allocated PERCORE page into user CR3 creation path.

Result:
- x64 user SYSCALL path became stable.

Stage 3: Align interrupt-gate entry with the same per-thread entry stack
------------------------------------------------------------------------
Problem:
- User-mode interrupts and exceptions enter through IDT gates, not through
  SYSCALL.
- Their hardware stack switch depends on TSS.RSP0.

Solution:
- Keep TSS.RSP0 synchronized with PERCORE.kernel_rsp during scheduling.
- Initialize CPU0 kernel thread so PERCORE.kernel_rsp and TSS.RSP0 match.

Result:
- User-mode timer IRQ / exception entry uses the same current-thread kernel
  entry stack model as SYSCALL.

Stage 4: Clarify the thread context model
-----------------------------------------
Rejected direction:
- Temporarily writing interrupted user-mode state back into
  `ThreadBlock::context` from syscall / interrupt entry looked attractive, but
  it mixed two different meanings into one structure:
  - scheduler continuation
  - user-mode return frame

Final direction:
- The kernel stack is the authoritative storage for nested syscall / interrupt
  entry frames, much closer to the Linux model.
- `ThreadBlock::context` is used only as the scheduler continuation context.
- New threads are a special first-run case:
  - build a clean Ring3 entry context directly
  - let the existing x86/x64 Ring3 IRET/IRETQ path enter user mode

Result:
- The model became simpler and less self-contradictory.
- Temporary helper paths such as saving syscall user state back into `context`
  were removed again.

Important lessons
-----------------
1. x64 SYSCALL and interrupt-gate entry are separate mechanisms.
   They must share a consistent kernel-entry-stack model, but they do not
   share the same hardware entry semantics.

2. GS pairing must be handled at every user <-> kernel boundary.
   Fixing only the SYSCALL entry path is not enough.

3. The hardware interrupt frame is the source of truth for interrupted
   user-mode state. Do not try to reconstruct it later from a scheduler-side
   kernel frame.

Current status
--------------
- PERCORE restored and in use.
- x64 SYSCALL path works with GS + PERCORE.
- PERCORE is mapped in user page tables.
- TSS.RSP0 tracks the current thread's entry stack.
- The kernel stack is the source of truth for nested user trap/syscall frames.
- `ThreadBlock::context` is used for scheduler continuation only.

Next recommended work
---------------------
- Add more debug assertions around PERCORE consistency:
  - current_thread != null
  - kernel_rsp != 0
  - tss.RSP0 == kernel_rsp
- Stress-test:
  - many syscalls
  - timer preemption under shell / cot
  - user faults such as page fault and general protection
- Then continue with the remaining x64 multithreading functionality on top of
  this verified base.

Thread-local ring0 stack window remap
-------------------------------------
Problem:
- After introducing TNEW, each thread correctly received its own
  `stack_levladdr`, but the fixed ring0 stack window `_NORMAL_RINGSTACK`
  inside a process page table still pointed only to `main_thread`.
- As a result, multiple threads inside the same process ended up reusing the
  same kernel-stack virtual window for scheduler continuations.
- A typical failure pattern was:
  - ThA -> ThB looked fine
  - switching back to ThA crashed in kernel mode with garbage CS/IP
  - the new thread entry itself was not the real problem

Why remapping was needed
------------------------
- The current scheduler/context-switch design stores kernel continuation stack
  pointers as linear addresses inside the fixed `_NORMAL_RINGSTACK` window.
- `PERCORE.kernel_rsp` and `TSS.RSP0` also currently use that same linear
  window model.
- Therefore, simply remembering a thread's physical ring0 stack is not enough:
  the CPU still dereferences RSP/ESP as a linear address under the current CR3.
- To keep the existing continuation model working, the fixed window must point
  at the incoming thread's `stack_levladdr` before restoring its kernel
  continuation.

Fix:
- Rebind `_NORMAL_RINGSTACK` to `new_tb->stack_levladdr` during scheduling.
- Update:
  - the currently active CR3
  - the target process page table
  - kernel_paging
- Flush the affected TLB entries with `invlpg` (`RefreshVirtualAddress()`).

Engineering note:
- This remap + invlpg model is a valid short/mid-term solution and fixes the
  current corruption problem.
- It should be treated as an implementation bridge, not necessarily the final
  architecture.
- A cleaner long-term design would give each thread a stable kernel virtual
  stack address and stop relying on a shared fixed window.

POSIX thread syscall migration notes
------------------------------------
Recovered upper-layer thread support on top of the repaired low-level base in
small verified steps:

1. Runtime thread logic was moved into `mecocoa/threads.cpp`
   instead of keeping TNEW/TEXI/TJOI/TDET/FUTX inside `syscall.cpp` or
   `taskman-new.cpp`.

2. `TNEW`
   - New threads no longer inherit the parent's transient syscall context.
   - Instead, build a clean first-run Ring3 context directly:
     - IP / SP / CS / SS / FLAGS / CR3
     - x86 stack argument + fake return
     - x64 RDI argument + fake return slot

3. `TEXI`
   - Main thread still exits through process exit logic.
   - Worker threads become `Hanging`, are removed from ready queues, and then
     schedule away.

4. `TJOI`
   - Minimal join semantics implemented with a per-thread `join_wait_queue`.
   - Join returns the worker exit status and reaps the target thread.

5. `TDET`
   - Minimal detach semantics implemented with `is_detached`.
   - Detached threads are not joinable; already-dead detached threads are
     reaped immediately.

6. `FUTX`
   - Implemented only the minimum wait/wake pair required by
     `pthread_mutex_lock` and `pthread_mutex_unlock`.
   - Scope is intentionally small and process-local for now.

Smoke tests
-----------
During bring-up, `subapps/test.cpp` temporarily hosted three small pthread
smoke modes:
- create + join
- create + detach
- create + mutex/futex contention

These are meant to be toggled independently so regressions can be isolated
quickly without mixing several thread features at once.

After the pthread lifecycle and futex path stabilized, these thread-specific
smoke tests were moved to:
- `subapps/_test/thread.cpp`

`subapps/test.cpp` was then restored to its regular GUI/form test role so it
can continue to serve as a general integration test entry without being tied
to pthread bring-up work.

Final pthread edge cleanup
--------------------------
After the main lifecycle tests passed, two small but important semantic fixes
were added:

1. `FUTEX_WAIT` race hardening
   - Re-check the futex value after taking the scheduler lock and only then
     enqueue the waiter.
   - This closes the classic wake-before-sleep window in the minimum process-
     local futex model.
   - `FUTEX_WAKE` now ignores `val == 0` and only wakes threads that are
     actually in `Pended + BR_Waiting` state for the requested futex address.

2. Detached thread auto-reap
   - A detached worker that exits becomes `Hanging` and schedules away as
     before, but is now destroyed automatically on the next scheduler pass
     after it is no longer executing.
   - This avoids accumulating finished detached kernel threads indefinitely
     while still keeping destruction off the thread's own active kernel stack.





