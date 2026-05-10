
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




