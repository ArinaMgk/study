
- 在 x86 多核架构中，传统的 8259A PIC 在物理连线上**只连接到了 CPU 0 (BSP)** 的 INTR 引脚上。AP 核（CPU 1, 2, ...）的外部中断引脚并没有与 8259A PIC 连接。


# 20260603

```
现在有四个核 cpu0 是 BSP，cpu 1cpu2 正常停机，为什么cpu3 还是

EIP=0000000c EFL=00010046 [---Z-P-] CPL=0 II=0 A20=1 SMM=0 HLT=0

ES =0000 00000000 0000ffff 00009300

CS =f000 000f0000 0000ffff 00009b00
```

这是一个在多核拓扑枚举中非常经典的“漏网之鱼”现象。

既然 CPU 1 和 CPU 2 都能极其完美地进入 `CS=0800, HLT=1` 的驻留状态，说明你的 `INIT-SIPI-SIPI` 广播序列、16 位跳板代码以及低位物理内存的 `cli` 已经无懈可击。而 CPU 3 依然顽固地保持着 `CS=f000, EIP=0000000c` 的通电复位状态（甚至 `HLT=0` 还在执行固件的 NOP 循环），这意味着：**CPU 3 压根没有收到主核心（BSP）发出的唤醒信号，或者在收到信号时由于时序冲突被硬件给静默丢弃（Dropped）了。**

结合 QEMU 模拟现代处理器的总线特性，导致 CPU 3 被独自落下的原因通常由以下三个底层细节引起：

### ## 1. x2APIC 广播简写（Shorthand=3）的硬件竞争陷阱

在你的 IPI 发送指令中，你使用了低 32 位的 `0xC4500` 和 `0xC4600`，其中的 `Bits [19:18] = 0b11 (3)` 代表 **"All Excluding Self"（除自己外的全系统广播）**。

- **硬件微观时序冲突：** 当你对 ICR 寄存器写入广播命令时，QEMU 的虚拟中断总线会**同时**向 CPU 1、CPU 2 和 CPU 3 派发中断。CPU 1 和 CPU 2 的虚拟线程响应极快，立刻进入了初始化流程。然而，由于你的 `PIT_Wait_MS_Direct(1)` 或者是 `TSC_Wait_MS` 延迟在 QEMU 软件模拟下**时间压得太死**，当 BSP 紧接着发出第一个和第二个 `SIPI` 时，CPU 3 内部的局部 APIC 状态机可能刚好卡在处理 `INIT` 信号的尾声，导致它的硬件接收缓冲区（IRR）发生拥堵，直接**静默丢弃**了紧随其后的 `SIPI`。
    
- **后果：** 错过了 `SIPI` 的 CPU 3 将永远停留在它的 Reset 固件环里，再也无法跨进你的 `0x8000` 大门。
    

### ## 2. QEMU 的 `-smp` 核心拓扑声明不符合预期

请切回你的控制台，检查你当前启动 QEMU 的命令行参数。多核声明的格式非常敏感：

- **不推荐的写法：** 比如 `-smp 4`。有时候 QEMU 会将其隐式解释为 4 个独立的 Sockets（插槽），或者在非对称的拓扑下分配不连续的 Local APIC ID。
    
- **推荐的工业级写法：** 必须显式指定插槽和核心匹配关系，强迫 QEMU 分配连续且对等的硬件线程：
    
    Bash
    
    ```
    -smp cores=4,threads=1,sockets=1
    ```
    
    如果拓扑不连续，某些广播总线在模拟时会产生寻址空洞，导致靠后的核心（如 CPU 3）收不到广播。
    

### ## 🛠️ 干净利落的修正方案（TAB 缩进 + 英文注释）

要彻底制服 CPU 3 这种“动作慢”的核心，有两种在工业级内核中被广泛采用的改良方案：

#### 方案 A：拉长 SIPI 的缓冲时序（最有效、最简单）

既然是因为 QEMU 并行模拟时时序太赶，我们只需要在发送 `INIT` 后，以及在两个 `SIPI` 之间，**给总线留出更充裕的喘息时间**：

```
	if constexpr (true) {
		// Send INIT IPI to all other cores.
		setMSR64(x86MSR::IA32_X2APIC_ICR, 0x00000000_000C4500ULL);
		TSC_Wait_MS(15); // Increase buffer from 10ms to 15ms for sluggish emulator threads.

		uint64_t sipi_command = 0x00000000_000C4600ULL | (ap_entry >> PAGESIZE_4KB);
		
		// Send the first SIPI
		setMSR64(x86MSR::IA32_X2APIC_ICR, sipi_command);
		TSC_Wait_MS(2);  // Give the bus a full 2ms to propagate the first wave.

		// Send the second SIPI to catch any cores that missed the first one (like CPU 3).
		setMSR64(x86MSR::IA32_X2APIC_ICR, sipi_command);
		TSC_Wait_MS(5);  // Final stabilization pause.
	}
```

#### 方案 B：由广播降级为“精准点对点点名”（极度稳健）

如果拉长时序后 CPU 3 偶尔还是漏掉，说明广播简写在当前 QEMU 拓扑下存在竞争。你可以利用循环，根据各个核心的 **Local APIC ID** 进行精准的一对一唤醒。

在 x2APIC 模式下，当你向 ICR 写入中断时，只要不使用简写（Shorthand=0），**高 32 位（EDX）就代表目标核心的 x2APIC ID**：

```
	// Bulletproof Point-to-Point (Unicast) AP Bootstrapping Loop.
	// Instead of broadcasting, we wake up CPU 1, 2, and 3 individually by their x2APIC IDs.
	for (uint32_t target_apic_id = 1; target_apic_id < 4; target_apic_id++) {
		// Pack target x2APIC ID into the upper 32 bits (EDX)
		uint64_t dest_field = (static_cast<uint64_t>(target_apic_id) << 32);

		// 1. Send INIT directly to target core (Shorthand: 0, Delivery: 5)
		setMSR64(x86MSR::IA32_X2APIC_ICR, dest_field | 0x00004500ULL);
		TSC_Wait_MS(10);

		// 2. Send SIPI 1 to target core (Shorthand: 0, Delivery: 6)
		uint64_t sipi_cmd = 0x00004600ULL | (ap_entry >> PAGESIZE_4KB);
		setMSR64(x86MSR::IA32_X2APIC_ICR, dest_field | sipi_cmd);
		TSC_Wait_MS(1);

		// 3. Send SIPI 2 to target core
		setMSR64(x86MSR::IA32_X2APIC_ICR, dest_field | sipi_cmd);
		TSC_Wait_MS(1);
	}
```

### 检查现在的 x86_32 中断是只给 cpu0 还是全部分发？

**答案是：目前的配置只分发给 CPU 0 (BSP)。**

通过检查你的 codebase，具体依据如下：

1. **I/O APIC 的初始化路由设置**： 在 interrupt.cpp:131-134 的 Local APIC 初始化流程中，I/O APIC 的重定向表项（RTE）被这样初始化：
    
```
    if (!ioapic_set) {
        for (stduint i = 0x10; i < 0x40; i += 2)
            IO_Writ64(i, 0x10020 + ((i - 0x10) >> 1));
        ioapic_set = true;
        ...
    }
```

**总结**：I/O APIC 中所有的外设中断路由，其目标 Local APIC ID 在初始化时均被指定为了 `0`（即 CPU 0 / BSP）。因此，目前在 APIC/x2APIC 模式下，**所有的外部设备中断都只会投递给 CPU 0，AP 核不会收到任何分发的外设中断**。

---

下一步就不要走 multiboot2，直接做 BIOS 路径的 RSDP 发现。你这个场景下，正确顺序是：

1. 先找 RSDP。  
    BIOS 下标准做法是找这两个区域：
    - 先查 EBDA  
        从 BDA 的 0x40E 取出 EBDA segment，左移 4 得到线性地址。  
        在 EBDA 起始后的前 1KB 里按 16 字节对齐扫描 "RSD PTR "。
        
    - 如果 EBDA 没找到，再扫 0xE0000 ~ 0xFFFFF  
        也是按 16 字节对齐扫描 "RSD PTR "。
        
2. 找到后校验 RSDP。  
    至少做两件事：
    - 前 8 字节必须是 "RSD PTR "
    - ACPI 1.0 校验前 20 字节和为 0
    - 如果 revision >= 2，再校验完整 length 字节和为 0
3. 先按 ACPI 1.0/32 位路径处理。  
    你现在是 x86 32-bit BIOS，第一版建议：
    - 优先用 rsdp->RsdtAddress
    - 暂时先不依赖 XSDT

这样最简单，够你把 MADT 跑起来。

4. 解析 RSDT。  
    RSDT 头后面是一串 32 位表指针。遍历每个 SDT：
    - 看签名是不是 "APIC"
    - 找到 "APIC" 就拿到 MADT
5. 解析 MADT。  
    你现在只需要先处理最关键的一类条目：

- Type 0: Processor Local APIC

对每个条目：

- 取 apic_id
- 看 flags & 1 是否 enabled
- enabled 的就登记成一个 CPU

6. 用 MADT 结果填你刚做好的 per-core 表。  
    也就是：
    - PCU_CORES_PERCORE[i]->lapic_id = apic_id
    - g_lapicid_to_coreid[apic_id] = i
    - Taskman::PCU_CORES = actual_count
7. 先只把 BSP 枚举和打印做通。  
    在动 AP 汇编前，先确认你能打印出类似：

`RSDP found at ... RSDT found at ... MADT found at ... CPU0 lapic=00 CPU1 lapic=02 CPU2 lapic=04 ...`

这一步通了，后面 AP 才有查表基础。

---

================================================================================
Mecocoa 系统引导挂起问题深度诊断与修复分析报告
================================================================================

本文档详细记录了在系统引导过程中发现并修复的两大关键 Bug 的成因、工作原理、调度/中断机制以及最终的解决方案。

--------------------------------------------------------------------------------
一、 多核调度器就绪队列泄露与 Epoch 死锁问题 (Scheduler Queue Leak & Epoch Deadlock)
--------------------------------------------------------------------------------

1. 背景与现象
   在开启多核（AP 核心）调度的环境下，系统引导到一半时会挂起。
   - CPU 0 处于空闲（HLT），部分 AP 核心在 `serv_cons_loop` 或 `serv_graf_loop` 线程中空转。
   - 调度器的 Ready Queues（就绪队列）长期保持非空（bitmap 指示有就绪任务），但实际没有可调度的线程在队列中运行。

2. 核心原理与切换状态保护
   在多核调度中，`switching_out_threads` 和 `just_schedule` 起到了重要的并发同步保护作用：
   - `PCU_CORES_switching_out_threads[cpuid]` 记录每个 CPU 核心当前正在切出的线程（旧线程）。
   - 当调度器切出当前线程 `old_tb` 时，会将其 `just_schedule` 标志设为 `1`，并赋值给 `switching_out_threads`。
   - 实际的硬件/寄存器上下文切换发生在汇编代码中，在新的线程恢复执行后，新线程会将前一个被切换线程的 `just_schedule` 标记清零。
   - 唤醒 AP 的 BSP 函数 `WakeOneIdleApForReadyWork()` 会检查目标 AP 是否处于空闲：
     ```cpp
     Taskman::current_thread(next_ap) == Taskman::idle_thread(next_ap) &&
     (Taskman::switching_out_threads(next_ap) == nullptr ||
      !Taskman::switching_out_threads(next_ap)->just_schedule)
     ```
     这保证了只有当目标 AP 完全完成了前一次的上下文切换（即旧线程的状态和寄存器已被安全保存）之后，BSP 才会向其发送重调度 IPI（Reschedule IPI）。这避免了在上下文切换尚未完成时重入中断，防止了栈破坏和调度器重入。

3. Bug 成因分析 (就绪队列泄露)
   - 当一个运行中的线程 `old_tb` 的时间片用尽或被迫让出 CPU 时，调度器在 `Schedule()` 中会将其重新放回就绪队列（Ready Queue）或过期队列（Expired Queue），并将其状态更新为 `TBS::Ready`。
   - 接着，调度器调用 `PickNext()` 从优先级队列中寻找下一个可以执行的就绪线程 `new_tb`。
   - 如果此时没有其他就绪任务，`PickNext()` 返回 `nullptr`。调度器执行 Fallback 机制，决定让当前线程继续运行：`new_tb = old_tb`。
   - 调度器检测到 `new_tb == old_tb` 后，直接将状态设置为 `Running` 并退出返回：
     ```cpp
     if (new_tb == old_tb) {
         if (old_tb->state == TBS::Ready) old_tb->state = TBS::Running;
         // BUG: 缺少将 old_tb 从就绪/过期队列中摘除的代码！
         old_tb->processor_id = cpuid;
         scheduler_lock.Release(old_if);
         return;
     }
     ```
   - **后果**：由于未调用 `DequeueReady(old_tb, false)` 摘除节点，该线程在继续作为 `Running` 状态在 CPU 上运行的同时，其节点依然遗留在就绪/过期队列中。
   - 这引发了队列泄露。由于泄露的 Running 线程节点一直在队列中，优先级队列的就绪位图（Ready Bitmap）永远无法清空。
   - 因为就绪位图无法清空，调度器在 `PickNext()` 中永远无法触发 Active Queue 和 Expired Queue 的 Epoch 交换（Epoch Swap）。这直接导致被放入 Expired Queue 的其他关键系统任务（如 `serv_cons_loop`、`serv_graf_loop`）永久饿死，系统发生局部死锁并挂起。

4. 解决方案
   在 `Taskman::Schedule()` 的 `new_tb == old_tb` 分支中，显式调用 `Taskman::DequeueReady(old_tb, false)`，确保继续运行的线程不会泄露在就绪/过期队列中：
   ```cpp
   if (new_tb == old_tb) {
       if (old_tb->state == TBS::Ready) old_tb->state = TBS::Running;
       Taskman::DequeueReady(old_tb, false); // <--- 修复：摘除当前回退线程的队列节点
       old_tb->processor_id = cpuid;
       scheduler_lock.Release(old_if);
       return;
   }
   ```

--------------------------------------------------------------------------------
二、 软盘驱动中断死锁问题 (Floppy Disk Interrupt Deadlock)
--------------------------------------------------------------------------------

1. 背景与现象
   在修复调度器队列泄露后，系统引导仍然卡死，控制台无输出。分析 CPU 和线程状态表发现：
   - 线程 `serv_file_loop` (TID 4) 阻塞在向软盘服务进程发送消息（`Send` 阻塞，目标为 TID 7）。
   - 软盘服务线程 `serv_dev_fl_loop` (TID 7) 处于 `Pended` 状态，阻塞在 `Recv` 中断信号上（等待 `INTRUPT` 唤醒）。

2. 软盘中断锁定与反馈机制
   - 软盘驱动程序通过一个中断自旋锁/标志量 `flp_lock` 来管理 IRQ 6（软盘中断）的处理：
     - `static byte flp_lock = 1;` 默认为锁定状态。
     - 在软盘中断处理程序 `Handint_FLP()` 中：
       ```cpp
       void Handint_FLP() {
           if (flp_lock) {
               IC.SendEOI(IRQ_Floppy);
               return; // 丢弃中断，不唤醒服务线程
           }
           flp_lock = 1; // 重新上锁
           rupt_proc(Task_Flp_Serv, IRQ_Floppy); // 唤醒软盘服务进程
           IC.SendEOI(IRQ_Floppy);
       }
       ```
     - 只有当 `flp_lock == 0` 时，中断处理程序才会唤醒软盘服务线程；否则，中断会被直接丢弃。
   - 驱动的 `Read()` 和 `Write()` 会在发起 I/O 指令前，调用操作系统绑定的反馈回调函数 `asserv(fn_feedback)();`（对应 `flp_rw_foreback()`），该回调会将 `flp_lock` 设为 `0`，从而使得之后触发的软盘传输完成中断能够成功解除锁定并唤醒线程。

3. Bug 成因分析 (中断丢失死锁)
   - 在 `unisym` freestanding 设备库的 [Floppy.cpp](file:///d:/her/unisym/lib/cpp/Device/Storage/Floppy.cpp) 中，`Recalibrate()` 和 `IsMediaPresent()` 两个函数均会向软盘控制器发送重校准命令（`FDC_CMD_RECALIBRATE`），并通过 `fn_int_wait()` (挂起当前线程) 来同步等待中断触发。
   - **致命漏洞**：与 `Read()` 和 `Write()` 不同，`Recalibrate()` 和 `IsMediaPresent()` 在发送命令前，**没有调用** `fn_feedback` 接口来清除 `flp_lock`。
   - 引导过程中，`Reset()` 初始化软盘控制器时，在内部手动设置了 `flp_lock = 0`，因此第一次 reset 中断和紧接着的第一次 recalibrate 中断可以成功进入。但第一次校准中断触发后，`Handint_FLP` 自动将 `flp_lock` 设回了 `1`。
   - 当 `Reset()` 结束后，`serv_dev_fl_loop()` 立即对驱动器调用 `IsMediaPresent()` 来检测软驱中是否有介质。
   - `IsMediaPresent()` 向控制器发送了 `FDC_CMD_RECALIBRATE` 命令，并调用 `fn_int_wait()` 阻塞等待。
   - 然而，由于 `IsMediaPresent()` 内部并未清除 `flp_lock`，`flp_lock` 依然为上次中断遗留下来的 `1`。
   - 随后软盘物理完成重新定位并触发中断，但在中断处理器中检测到 `flp_lock == 1`，中断被直接丢弃。
   - 软盘服务线程（TID 7）永远等不到中断唤醒，陷入无限等待状态。其他文件系统线程（如 TID 4）因向其发送同步消息而相继挂起，整个引导进程死锁。

4. 解决方案
   修改 `unisym` 库中的 [Floppy.cpp](file:///d:/her/unisym/lib/cpp/Device/Storage/Floppy.cpp)，在 `FloppyDisk::Recalibrate()` 和 `FloppyDisk::IsMediaPresent()` 触发校准命令并阻塞等待中断之前，注入 `asserv(fn_feedback)();` 回调：
   - 在 `Recalibrate()` 中：
     ```cpp
     void FloppyDisk::Recalibrate() {
         Motor(true);
         asserv(fn_feedback)(); // <--- 修复：调用反馈清零 flp_lock
         WriteCmd(FDC_CMD_RECALIBRATE);
         WriteCmd(id);
         if (fn_int_wait) fn_int_wait();
         ...
     }
     ```
   - 在 `IsMediaPresent()` 中：
     ```cpp
     bool FloppyDisk::IsMediaPresent() {
         ...
         // Force seek/recalibrate to clear potential false positive from recent disk swap
         asserv(fn_feedback)(); // <--- 修复：调用反馈清零 flp_lock
         WriteCmd(FDC_CMD_RECALIBRATE);
         WriteCmd(id);
         if (fn_int_wait) fn_int_wait();
         ...
     }
     ```
   这样，在进入等待之前 `flp_lock` 都会被成功重置为 `0`，中断信号得以顺利被捕获并唤醒软盘服务进程，从而完美解决引导死锁。
================================================================================

Stage 区新增函数与字段总结

说明
- 这里只总结“stage 区新增”的函数、字段、全局状态/常量。
- 只写作用，不评价对错。
- 按文件分组。

1. include/taskman.hpp

新增常量
- IRQ_RESCHED_IPI = 0xF1
  作用：x8632 上给 AP 发送“重新调度”IPI 的中断向量号。
- IRQ_WAKE_IPI = 0xF2
  作用：x8632 上给 AP 发送“唤醒”IPI 的中断向量号。

新增全局声明
- ap_ring3_iret_stack_tops[PCU_CORES_MAX]
  作用：保存每个 CPU 的 ring3 IRET scratch stack 顶地址。
- ap_resched_ipi_hits[PCU_CORES_MAX]
  作用：统计每个 CPU 收到 RESCHED IPI 的次数，主要用于调试。
- ap_ring3_iret_guard_hits
  作用：统计 ring3 IRET stack guard/fallback 命中的次数，主要用于调试。
- ap_ring3_iret_last_lapicid
  作用：记录最近一次 ring3 IRET guard 触发时看到的 LAPIC ID，主要用于调试。
- ap_ring3_iret_last_coreid
  作用：记录最近一次 ring3 IRET guard 触发时映射到的 core id，主要用于调试。

新增函数声明
- SendWakeAllApsIPI()
  作用：由 BSP 向所有 AP 广播 WAKE IPI。
- WakeOneIdleApForReadyWork()
  作用：在 ready work 足够时唤醒一个空闲 AP 参与调度。
- GetCoreRingStackBase(stduint cpuid)
  作用：根据 CPU id 计算该核专用的 ring0 高地址栈窗口基址。
- Taskman::SendRescheduleIPI(stduint core_id)
  作用：向指定 AP 发送 RESCHED IPI。
- Taskman::SendWakeIPI(stduint core_id)
  作用：向指定 AP 发送 WAKE IPI。

新增字段
- PERCORE::tss_selector
  作用：x8632 下记录每个 CPU 自己的 TSS 选择子，供 loadTask/LTR 使用。
- ThreadBlock::name
  作用：给线程挂一个只读名字，便于日志和调试输出。
- ThreadBlock::ring_coreid
  作用：记录线程当前被限制继续运行的 core id；~0 表示不限制。
- ThreadBlock::first_run_cpu
  作用：记录线程第一次真正被调度运行的 CPU。
- ThreadBlock::has_run_once
  作用：标记线程是否已经至少运行过一次。

2. mecocoa/schedul.cpp

新增全局状态
- ap_lapicid_to_coreid[LAPIC_ID_MAP_SIZE]
  作用：x8632 下 LAPIC ID 到内核 CPU 编号的映射表。
- ap_higher_stack_tops[PCU_CORES_MAX]
  作用：x8632 下每核高地址内核栈窗口的栈顶地址。
- ap_ring3_iret_stack_tops[PCU_CORES_MAX]
  作用：x8632 下每核 ring3 IRET scratch stack 的栈顶地址。
- ap_resched_ipi_hits[PCU_CORES_MAX]
  作用：记录各核收到 RESCHED IPI 次数。
- ap_ring3_iret_guard_hits
  作用：记录 ring3 IRET 栈 guard 的命中次数。
- ap_ring3_iret_last_lapicid
  作用：记录最近一次 guard 时的 LAPIC ID。
- ap_ring3_iret_last_coreid
  作用：记录最近一次 guard 时的 core id。
- g_enable_ap_scheduling
  作用：总开关，控制 AP 是否真的参与调度。

新增函数
- ReleaseSchedulerLockForSwitch()
  作用：切换前只释放 scheduler_lock，不通过常规 Release 路径恢复其它状态。
- InitializeLocalApicOnAp()
  作用：在 AP 上初始化本地 APIC/x2APIC，并屏蔽部分 LVT，中断 EOI 清理。
- HandleRescheduleIPI()
  作用：AP 处理 RESCHED IPI；统计命中、EOI，并按开关进入 Schedule(true)。
- HandleWakeIPI()
  作用：AP 处理 WAKE IPI；只 EOI。
- WakeThreadOnRecordedCpu(ThreadBlock* th)
  作用：按线程的 processor_id，把对应 CPU 从 HLT 中唤醒。
- CountQueuedReadyTasks(stduint limit)
  作用：粗略统计 ready/expired 队列里 Ready 线程数量，到达上限即提前返回。
- WakeOneIdleApForReadyWork()
  作用：由 BSP 选择一个 idle AP，发送 RESCHED IPI，让它来拿 ready work。
- AP_Main(stduint core_id)
  作用：x8632 AP 启动入口；设置 idle/current、绑定内核栈、装 APIC/IDT/TSS/LDT，然后进入 HLT 循环。
- SettleSwitchingOutThread(stduint cpuid)
  作用：在调度点收尾之前上一条 switching_out 线程；必要时销毁 detached hanging 线程。

新增/扩展后的行为相关点
- BindCurrentKernelEntryStack(ThreadBlock* th, stduint cpuid) [x8632 分支增强]
  作用：把指定线程的 ring0 栈重新绑定到当前 CPU 的固定高地址窗口，并刷新 TSS.ESP0。
- RebindThreadKernelStackWindow(ThreadBlock* th) [增强]
  作用：按当前 CPU 的专属高地址窗口重新映射线程 ring0 栈，而不是固定只用一个全局窗口。

3. mecocoa/taskman.cpp

新增函数
- Taskman::getID() [x8632 实现]
  作用：通过 CPUID 0x0B 读取 LAPIC ID，再映射成内核 core id。
- HandleRescheduleIPI()
  作用：BSP 侧注册的 RESCHED IPI handler；当前实现只 EOI。
- HandleWakeIPI()
  作用：BSP 侧注册的 WAKE IPI handler；当前实现只 EOI。
- SendFixedIPIByCore(stduint core_id, uint8 vector)
  作用：按目标 core 的 LAPIC ID 发送 fixed IPI，支持 xAPIC/x2APIC。
- Taskman::SendRescheduleIPI(stduint core_id)
  作用：发送 RESCHED IPI 到指定 AP。
- Taskman::SendWakeIPI(stduint core_id)
  作用：发送 WAKE IPI 到指定 AP。
- SendWakeAllApsIPI()
  作用：向所有 AP 广播 WAKE IPI。

新增/扩展字段初始化
- Taskman::AllocateThread() 中给 tb->processor_id = CORE_ID_INVALID
  作用：新线程默认不归属于任何 CPU，避免误判成 CPU0。

4. mecocoa/taskman-new.cpp

新增全局
- higher_stacks[PCU_CORES_MAX]
  作用：每核的高地址内核栈 backing 页。
- ring3_iret_stacks[PCU_CORES_MAX] [x8632]
  作用：每核的 ring3 IRET scratch stack backing 页。

新增/增强函数
- _Mapping_Core_Stack(Paging& paging) [x8632 分支增强]
  作用：把每核 ring3 IRET stack 映射到各自固定虚拟地址页。
- _Taskman_Create_Paging(...) [多处增强]
  作用：
  1. 为每个 CPU 映射独立的 ring0 高地址窗口。
  2. 初始化每核 higher_stacks / ring3_iret_stacks。
  3. 初始化每核 LAPIC/core 映射。
  4. 为每个 AP 分配自己的 TSS selector。
  5. 初始化 kernel thread 的 processor_id / ring_coreid / name。
  6. 给 idle 线程统一设置 pid=-1、tid=-1，避免和 TID0/PID0 混淆。

新增字段赋值
- kernel_thread->processor_id = cpuid
  作用：把内核主线程标记为当前 CPU 所属。
- kernel_thread->ring_coreid = cpuid
  作用：把内核主线程限制在当前 CPU。
- kernel_thread->name = "kernel"
  作用：给内核主线程命名。
- idle_task->pid = ~0
  作用：idle 任务不占用普通 PID 编号。
- idle_task->main_thread->tid = ~0
  作用：idle 线程不占用普通 TID 编号。

5. mecocoa/syscall.cpp

新增逻辑相关字段使用
- Handint_SYSCALL(CallgateFrame* frame) 中新增：
  - crt_th
  - was_pinned
  - cpu_id
  作用：
  1. 进入 syscall 时，按当前 CPU 给线程设置/维持 ring_coreid。
  2. syscall 尾部在适当条件下清除 ring_coreid。

3. mecocoa/handler.cpp

新增函数
- FrameFromUser(const HardwareInterruptFrame* frame)
  作用：判断当前中断/异常帧是否来自用户态。
- FrameSavedEsp(const HardwareInterruptFrame* frame)
  作用：统一取“应当显示的 ESP”，避免把 ring0 帧误当成用户态 ESP。
- FrameSavedSs(const HardwareInterruptFrame* frame)
  作用：统一取“应当显示的 SS”。
- LogSelectorFaultContext(rostr name, HardwareInterruptFrame* frame, stduint para)
  作用：x8632 下对 #TS/#NP/#SS/#GP 这类 selector fault 打更详细的 CPU/TID/CS/SS/CR3/LDT-GDT-IDT 诊断。

新增调试外部变量引用
- ap_ring3_iret_guard_hits
- ap_ring3_iret_last_lapicid
- ap_ring3_iret_last_coreid
  作用：在异常日志里带出 ring3 IRET scratch stack 相关调试信息。

7. mecocoa/sysinfo.cpp

新增函数
- dump_threads(OstreamTrait& com1)
  作用：列出线程链中的线程信息，包括状态、原因、CPU、优先级、IP/SP、消息队列关系、名字等。
- dump_processors(OstreamTrait& com1)
  作用：列出各 CPU 的状态、LAPIC、current_thread、switching_out_thread、kernel stack。
- dump_ready_queue(OstreamTrait& com1)
  作用：列出 active ready queue 与 expired ready queue 的内容。

8. devdriv/timer/timer-pit.cpp

新增调用点
- time_slice == 2 时调用 SendWakeAllApsIPI()
  作用：BSP 定期广播 WAKE，把睡着的 AP 弹起来。
- time_slice >= 4 时调用 WakeOneIdleApForReadyWork()
  作用：BSP 在调度点前尝试叫醒一个 idle AP 分担 ready work。

9. devdriv/tty/serial.cpp

新增调用点
- COM1 输入字符 's' 时调用：
  - dump_processors()
  - dump_threads()
  - dump_ready_queue()
  作用：串口快速触发调度/线程状态转储。

10. include/archits/atx-x86-flap32.hpp / prehost/atx-x86-flap32/atx-x86.asm

新增中断入口声明/桩
- Handint_RESCHED_Entry
  作用：RESCHED IPI 的汇编入口。
- Handint_WAKE_Entry
  作用：WAKE IPI 的汇编入口。


这次最核心的新机制（便于整体理解）

- ring_coreid
  作用：在特定窗口里限制线程只能由某个 CPU 继续调度。

- processor_id
  这次不是新字段，但本次被大量新逻辑使用。
  作用：记录线程最近运行在哪个 CPU，并作为 WAKE 的目标依据。

- ap_* 系列状态
  作用：支撑 x8632 AP bring-up、每核栈窗口、IPI 统计、ring3 IRET 调试。

- SendWakeAllApsIPI / SendWakeIPI / SendRescheduleIPI
  作用：形成 x8632 上 BSP -> AP 的两类动作：
  1. WAKE：只把 AP 从 HLT 弹醒。
  2. RESCHED：让 AP 进入调度器抢活。











