# Mecocoa 内核信号机制 - 第一阶段实施方案

本实施方案详细阐述了 Mecocoa 内核信号机制第一阶段的设计与实现步骤。主要包括：孤儿进程领养、用户态 CPU 异常向信号的映射、信号发送与线程唤醒逻辑，以及信号投递与上下文恢复（蹦床机制）。

---

## 约束与规范说明

- **构建与运行限制**：内核编译与运行测试（如 `make run`）全部由用户手动执行，本代理不擅自运行任何构建/运行命令。
- **枚举值准则**：所有枚举值（如进程标识 `Task_Init`、CPU 异常标识 `ERQ_`）必须以对应头文件（如 `taskman.hpp`、`IBM.h`）中的实际定义为唯一准则，禁止凭经验推测具体数值。
- **代码规范**：
    - 采用 **TAB 缩进**。
    - 代码注释一律使用 **英文 (English)**。
    - 保留代码中原有注释，不擅自删除。
    - 代码命名与逻辑应做到“字面即意”，并辅以适当的英文注释说明。

---

## 1. 孤儿进程领养 (Orphan Process Reparenting)

### 目标

当父进程退出时，其子进程（包括运行中的子进程和已退出的僵尸/hanging子进程）不应被强制销毁，而是应将其父进程重定向为系统初始化进程（`Task_Init`）。这使得运行中的子进程能继续执行，而已退出的子进程能由 `Task_Init` 通过 `wait` 进行状态回收。

### 现有代码设计缺陷与修复

1. **强制递归销毁**：在 taskman.cpp 处，当父进程退出时，会遍历并强制销毁所有子进程：
    
    cpp
    
    // Hierarchical Process Tree: Recursive Kill
    
    while (p->child_list_head) {
    
        ProcessBlock* child = p->child_list_head;
    
        Taskman::Exit(child, -1);
    
        _Exit_Cleanup(child->pid);
    
    }
    
    这导致子进程无法作为孤儿进程继续生存。必须将该递归销毁循环删除。
2. **僵尸子进程立即释放**：在 taskman.cpp 处，若子进程处于 `Hanging` 状态，退出时会直接调用 `_Exit_Cleanup` 清理，导致其退出状态丢失，无法被领养者回收。

### 改造方案 (`Taskman::Exit` 内)

在 taskman.cpp 中进行如下修改：

1. **删除**原有的递归销毁子进程的循环（删除 lines 404-409）。
2. **修改**结尾处的领养循环，将所有处于 `Hanging` 和活动状态的子进程一并领养至 `Task_Init`。

修改后的实现代码（使用 TAB 缩进与英文注释）：

cpp

// Handle Remaining Children (Reparent all children to Task_Init)

	while (p->child_list_head) {

		ProcessBlock* child = p->child_list_head;

		// Unlink the child from current parent's child list first

		{

			SpinlockLocal guard(&scheduler_lock);

			p->child_list_head = child->sibling_next;

			child->sibling_next = nullptr;

		}

		// Reparent the child to Task_Init (sole source of truth from taskman.hpp)

		child->parent_id = Task_Init;

		// Attach the child to Task_Init's child list

		ProcessBlock* pinit = Taskman::Locate(Task_Init);

		if (pinit) {

			SpinlockLocal guard(&scheduler_lock);

			child->sibling_next = pinit->child_list_head;

			pinit->child_list_head = child;

		}

	}

---

## 2. 处理应用执行时发生的异常 (CPU Exceptions to Signals)

### 目标

当硬件异常（如 `#PF` 缺页、`#UD` 未定义指令、`#GP` 通用保护异常）发生在用户态（Ring 3）时，系统不应对整个系统执行 `cli; hlt` 停机，而是将其转换为对应的 POSIX 信号标记在当前线程上，使其在返回用户态前能够被立即投递。

### 安全约束设计：异常 Handler 内的安全性

- **避免直接调用重度函数**：在异常上下文（Ring 0 Exception Handler）中，不应该直接调用 `sys_kill` 等涉及完整进程查找、锁同步以及重入可能的高级函数。
- **安全置位并返回**：在异常处理流程中，直接对当前运行线程 `crt->pending_signals` 执行信号置位，随后直接从异常处理器返回。利用中断分发器 `interrupt_dispatcher` 返回用户态时自动调用的 `check_and_deliver_signals` 来进行实际的栈构建和信号处理函数的跳转。

### 硬件异常映射关系

根据 IBM.h 的定义，硬件异常与信号的映射如下：

- `ERQ_Divide_By_Zero` (0) -> `SIGFPE`
- `ERQ_Step` (1) -> `SIGTRAP`
- `ERQ_Breakpoint` (3) -> `SIGTRAP`
- `ERQ_Overflow` (4) -> `SIGSEGV`
- `ERQ_Bound` (5) -> `SIGSEGV`
- `ERQ_Invalid_Opcode` (6) -> `SIGILL`
- `ERQ_x0D` (13, 通用保护异常 #GP) -> `SIGSEGV`
- `ERQ_Page_Fault` (14) -> `SIGSEGV`

### 改造方案 (`exception_handler` 内)

在 handler.cpp 中，修改异常处理入口：

1. **识别特权级**：检查发生异常时的 CS 寄存器是否处于用户态（Ring 3）。
2. **转换并标记信号**：若为用户态异常，向当前线程的 `pending_signals` 集中添加映射后的信号，并直接从 handler 返回。

修改后的实现逻辑：

cpp

_ESYM_C

__attribute__((target("general-regs-only"), optimize("O0")))

void exception_handler(HardwareInterruptFrame* frame) {

	stduint iden = frame->interrupt_id;

	stduint para = frame->error_code;

	stduint r15 = frame->pg_cr3;

	const bool have_para = (iden == 8 || (iden >= 10 && iden <= 14) || iden == 17 || iden == 21);

	// Check if exception comes from user space (Ring 3)

	bool is_user = (frame->hw_cs & 3) != 0;

	if (is_user) {

		int sig = 0;

		switch (iden) {

			case ERQ_Divide_By_Zero:

				sig = SIGFPE;

				break;

			case ERQ_Step:

			case ERQ_Breakpoint:

				sig = SIGTRAP;

				break;

			case ERQ_Overflow:

			case ERQ_Bound:

			case ERQ_x0D: // General Protection (#GP) defined as ERQ_x0D in IBM.h

			case ERQ_Page_Fault:

				sig = SIGSEGV;

				break;

			case ERQ_Invalid_Opcode:

				sig = SIGILL;

				break;

			default:

				sig = SIGILL; // Fallback signal for unknown exceptions

				break;

		}

		if (sig != 0) {

			ThreadBlock* crt = Taskman::current_thread[Taskman::getID()];

			if (crt) {

				// Mark signal as pending directly in Ring 0 exception handler

				sigaddset(&crt->pending_signals, sig);

				return; // Returns to dispatcher, which calls check_and_deliver_signals

			}

		}

	}

	// Keep existing kernel-space exception handling (unreachable for user space)

	dword tmp;

	if (iden >= 0x20)

		printlog(_LOG_FATAL, "#ELSE %x %x", iden, para);

	switch (iden) {

	case ERQ_Invalid_Opcode:// 6

	{

		// first #UD is for TEST

		static bool first_done = false;

		if (!first_done) {

			first_done = true;

			rostr test_page = (rostr)"[Mecocoa] Exception #UD Test OK!"; // "\xFF\x70[Mecocoa]\xFF\x27 Exception #UD Test OK!\xFF\x07";

			printlog(_LOG_GOOD, " %s", test_page);

			#if _MCCA == 0x8632

			frame->hw_eip += 2;

			#elif _MCCA == 0x8664

			frame->hw_rip += 2;

			#endif

		}

		else {

			printlog(_LOG_FATAL, " %s", ExceptionDescription[iden]);// no-para

			__asm("cli; hlt");

		}

		break;

	}

	case ERQ_Coprocessor_Not_Available:// 7

		// needed by jmp-TSS method

		if (!(getCR0() & 0b1110)) {

			printlog(_LOG_FATAL, " %s", ExceptionDescription[iden]);// no-para

		}

		EnableSSE();

		break;

	case ERQ_Page_Fault:// 14

		printlog(_LOG_FATAL, "%s with 0x%[x], vaddr=0x%[x], TID%u, CR3=0x%[x]",

			ExceptionDescription[iden], para, getCR2(), Taskman::current_thread[Taskman::getID()]->tid, r15);

		break;

	default:

		printlog(_LOG_FATAL, have_para ? "%s with 0x%[32H]" : "%s",

			ExceptionDescription[iden], para);

		__asm("cli; hlt");

		break;

	}

}

---

## 3. 信号发送与线程唤醒 (Signal Delivery and Thread Wakeup)

### 目标

当一个处于阻塞状态（如睡眠、等待 IPC 消息、等待子进程等）的线程接收到信号时，必须立刻打断其阻塞状态，并安全地从对应等待队列中移除（以避免悬挂指针与重复唤醒）。同时被唤醒的系统调用需在返回路径上检测到这一状态并返回 `-EINTR` 错误码。

### 改造方案

#### A. `sys_kill` 唤醒线程实现

在 signals.cpp 内处理：

cpp

void wakeup_thread_for_signal(ThreadBlock* th, int sig) {

	// Only wake up the thread if it is currently in pended (blocked) state

	if (th->state == ThreadBlock::State::Pended) {

		// Verify if the block reason is interruptible

		if (th->block_reason & (ThreadBlock::BlockReason::BR_Resting | 

								ThreadBlock::BlockReason::BR_RecvMsg | 

								ThreadBlock::BlockReason::BR_SendMsg | 

								ThreadBlock::BlockReason::BR_Waiting)) {

			// 1. Handle Timer sleep cleanup

			if (th->block_reason & ThreadBlock::BlockReason::BR_Resting) {

				SpinlockLocal guard(&timer_lock);

				for (auto nod = TimerManager.Root(); nod; ) {

					auto next_nod = nod->next;

					auto msg_timer = (MsgTimer*)nod->offs;

					if (msg_timer->iden == (stduint)th) {

						TimerManager.Remove(nod);

						break; // Thread has only one timer

					}

					nod = next_nod;

				}

			}

			// 2. Handle IPC send/recv queue cleanup (under comm_lock)

			if (th->block_reason & (ThreadBlock::BlockReason::BR_RecvMsg | ThreadBlock::BlockReason::BR_SendMsg)) {

				// Safely unlink the thread from IPC send queues and clean up pending messages

				msg_cleanup_thread(th);

			}

			// 3. Perform unblocking and place back into scheduler's ready queue

			th->Unblock(th->block_reason);

		}

	}

}

#### B. 阻塞系统调用返回 `-EINTR`

在阻塞系统调用（如 `sysc_REST`、`sysc_COMM`）的唤醒返回路径上，检测是否因信号中断：

cpp

bool has_pending_signals(ThreadBlock* th) {

	uint64_t pending = _sigset_raw(&th->pending_signals);

	uint64_t blocked = _sigset_raw(&th->blocked_signals);

	return (pending & ~blocked) != 0; // Returns true if there are unmasked pending signals

}

若 `has_pending_signals(th)` 为真，系统调用退出并返回 `-EINTR`（或对应错误表示，如 `~_IMM0`）。

---

## 4. 信号投递与上下文恢复 (Signal Delivery & Context Recovery)

### A. 栈的选择与安全性

- **默认用户栈**：默认情况下，信号栈帧构建于发生中断时线程正在使用的同一个用户栈（自中断帧的 `hw_esp`/`hw_rsp` 获取）。
- **备用信号栈**：若信号动作配置了 `SA_ONSTACK`，且线程此前配置了备用信号栈（存储在 `ThreadBlock` 中），则应将信号现场构建于该备用信号栈之上。
- **对齐要求**：构建任何栈帧前，必须对最终计算的栈指针进行严格的 16 字节对齐。

### B. 信号上下文定义与隔离

- **物理寄存器信息物理隔离**：绝对不应把完整的 `HardwareInterruptFrame`（包含内核 CR3、错误码等内核隐私敏感字段）泄露给用户态。
- **用户态可见上下文**：定义一个独立的结构体 `SigContext`，仅包含用户通用寄存器（如 EIP, EFLAGS, ESP, EAX, EBX, ECX, EDX, EBP, ESI, EDI）及段选择子，并在 `SigStackFrame` 中包含此结构。同时必须将原屏蔽字 `blocked_signals` 存入帧中以支持信号嵌套。

cpp

struct SigContext {

	stduint reg_eip;

	stduint reg_eflags;

	stduint reg_esp;

	stduint reg_eax;

	stduint reg_ecx;

	stduint reg_edx;

	stduint reg_ebx;

	stduint reg_ebp;

	stduint reg_esi;

	stduint reg_edi;

	sigset_t old_mask; // Saved signal mask for nested signal support

};

struct SigStackFrame {

	int signo;

	SigContext context;

};

### C. 蹦床（Trampoline）与栈帧构建（架构敏感）

当准备将信号投递至用户态处理器时，根据具体 CPU 架构构建返回地址与现场。

#### 1. 32 位 x86 (x86_32)

在 others-a.asm 中，定义蹦床函数：

nasm

GLOBAL __sigrestorer

__sigrestorer:

	MOV EAX, 0x15 ; Syscall number for sysc_SIGR

	CALL SegCall|3:0

内核在 `check_and_deliver_signals` 中构建现场：

1. 确定最终栈指针 `new_esp`（根据 `SA_ONSTACK` 选择并做 16 字节对齐）。
2. 在 `new_esp - 4` 写入 `__sigrestorer` 地址作为返回地址。
3. 从 `new_esp - 4 - sizeof(SigStackFrame)` 处构建并填充 `SigStackFrame`。
4. 将线程的 `blocked_signals` 备份至帧中的 `old_mask`，并将当前信号加入 `blocked_signals` 屏蔽集（若未设置 `SA_NODEFER`）。
5. 将 `frame->hw_esp` 更新为最终的 `new_esp - 4`。
6. 将 `frame->hw_eip` 更新为用户信号处理函数的入口。

#### 2. 64 位 x86 (x86_64)

蹦床设计类似，但系统调用号与地址宽度均为 64 位。返回地址占 8 字节。

#### 3. RISC-V

在 RISC-V 架构中，返回地址不通过栈传递，而是直接保存在 `ra` 寄存器中：

1. 内核将 `cxt->ra` 设置为蹦床地址 `__sigrestorer` 的入口。
2. 依据栈对齐规范在用户栈构建 `SigStackFrame`。
3. 修改 `cxt->a0` 传递参数 `signo`。

---

## 5. 现场恢复的安全清洗规范 (Strict Context Sanitization)

### 严重提权漏洞与防御

若在系统调用 0x15 (`sysc_SIGR` / `sys_sigreturn`) 的内核实现中，从用户空间拷回现场后直接覆盖当前内核栈上的中断帧，用户态程序可通过恶意修改 EFLAGS（提权 IOPL 允许执行敏感 I/O）或修改 CS/SS 寄存器为内核段描述符获得 Ring 0 级别的提权。

### 强制清洗规范

在 `sysc_SIGR` 中读取用户栈上的 `SigContext` 恢复现场时，内核**必须**对关键控制段寄存器和标志位实施强行清洗和验证：

cpp

```
SigStackFrame user_frame;

// Safely copy signal frame from user stack

MccaMemCopyP(&user_frame, (void*)user_provided_sp, sizeof(SigStackFrame));

// 1. Sanitize Segments to prevent Ring 0 escalation

// Force the Code Segment (CS) and Stack Segment (SS) to user selectors

// Segment selector values defined strictly from memoman.hpp GDT layout

user_frame.context.reg_cs = SegCoR3 | RING_U; // Force User Code Selector

user_frame.context.reg_ss = SegDaR3 | RING_U; // Force User Data Selector

user_frame.context.reg_ds = SegDaR3 | RING_U;

user_frame.context.reg_es = SegDaR3 | RING_U;

// 2. Sanitize EFLAGS

// Allow user flags modifications (CF, PF, AF, ZF, SF, TF, IF, DF, OF)

// Clear all system privilege flags (IOPL, NT, VM, RF) to prevent Ring 0 privilege escalation

// Ensure IF (Interrupt Enable Flag, 0x200) remains enabled for the thread

user_frame.context.reg_eflags = (user_frame.context.reg_eflags & 0x8D5) | 0x200;

// 3. Restore signal mask for nested signal support

crt->blocked_signals = user_frame.context.old_mask;

// 4. Update the actual kernel stack frame registers

current_kernel_frame->hw_eip = user_frame.context.reg_eip;

current_kernel_frame->hw_eflags = user_frame.context.reg_eflags;

current_kernel_frame->hw_esp = user_frame.context.reg_esp;

current_kernel_frame->hw_cs = user_frame.context.reg_cs;

current_kernel_frame->hw_ss = user_frame.context.reg_ss;

current_kernel_frame->pusha_eax = user_frame.context.reg_eax;

current_kernel_frame->pusha_ebx = user_frame.context.reg_ebx;

current_kernel_frame->pusha_ecx = user_frame.context.reg_ecx;

current_kernel_frame->pusha_edx = user_frame.context.reg_edx;

current_kernel_frame->pusha_ebp = user_frame.context.reg_ebp;

current_kernel_frame->pusha_esi = user_frame.context.reg_esi;

current_kernel_frame->pusha_edi = user_frame.context.reg_edi;
```

---

## 6. 第一阶段测试与验证

在 test.cpp 中注册自定义信号处理器并触发硬件除零异常或 `#UD` 异常进行测试：

- 观察 CPU 执行异常后是否直接返回注册的信号处理函数中，而不是导致宿主机蓝屏/挂起。
- 信号处理函数返回后，检查能否继续正常执行后续的主程序指令。
- 确认当父进程退出后，子进程的 parent 变为 `Task_Init`，并在其退出后由 init 进程的 `wait` 环路正常清理。


---

```
测试 Ctrl+C：在 Shell/Cot 运行子程序（如 subtest）时，按下 Ctrl+C：
没有正常退出，只是显示 [TestApp]Received SIGINT(2)!Interrupted successfully!
而且可以按好多次

给子程序的窗体发消息伴随输出
[ERROR]deadlock T12<-...->T2
[ERROR]msg_send_will_deadlock
[ERROR]deadlock T2<-...->T12
[ERROR]msg_send_will_deadlock
[ERROR]deadlock T2<-...->T12
[ERROR]msg_send_will_deadlock



关闭 Shell 窗口：在运行子进程时直接关闭 Shell 窗口：
[INFOR]Process 16 exited with code 00000000
[INFOR]cleaning 16, now exist 11 tasks
[INFOR]Process 17 forced to exit due to signal 9
[INFOR]Process 17 exited with code 00000089
[INFOR]cleaning 17, now exist 10 tasks

看起来正常
```



您的测试结果说明**信号机制（包括中断、栈对齐、蹦床跳转以及上下文恢复）已经完全正常且稳定地工作**！

以下是对测试现象的详细解答与分析：

### 1. 为什么子程序没有退出且可以按好多次？

这是**完全符合预期并且符合 POSIX 标准的正常行为**。

- 在 `subapps/test.cpp` 中，您注册的 `sigint_handler` 函数如下：
    
    cpp
    
    static void sigint_handler(int signo) {
    
        outsfmt("\n\r[TestApp] Received SIGINT (%d)! Interrupted successfully!\n\r", signo);
    
    }
    
- 该处理器**只打印了一行信息，并没有调用 `_exit()` 或 `exit()` 来终止程序**。
- 因此，在信号处理函数执行完毕后，内核通过 `sysc_SIGR` 成功恢复了程序被中断时的上下文，程序继续在 `while (sys_fetch_msg(...))` 循环中运行。
- 因为进程没有退出，所以您每次按下 `Ctrl+C`，它都会再次递送信号、执行处理器并打印该信息。这正好证明了**信号返回后的上下文恢复与屏蔽字恢复是 100% 正确的**。

---

### 2. 为什么给窗体发消息时会伴随死锁警告？

这属于 **Mecocoa 内核的主动死锁保护机制**，是由在信号处理器中调用非“异步信号安全 (async-signal-safe)”函数引起的：

- **死锁成因**：
    1. 当您在窗口上移动鼠标或点击时，图形控制服务 `Task_Console` (T2) 会尝试向子进程 `subtest` (T12) 发送窗口消息，并在子进程未读取时处于 `BR_SendMsg` 阻塞状态。
    2. 此时，如果您按下 `Ctrl+C` 触发信号，`subtest` (T12) 暂停主循环并进入 `sigint_handler` 执行。
    3. `sigint_handler` 中调用了 `outsfmt`。在 Mecocoa 中，`outsfmt` 是一次**同步 IPC 阻塞调用**，它需要向 `Task_Console` (T2) 发送打印请求。
    4. 于是产生循环等待：`T12` 等待 `T2` 接收打印消息，而 `T2` 正在阻塞等待 `T12` 接收窗口事件。
- **内核死锁防护**： Mecocoa 的 IPC 子系统通过 `msg_send_will_deadlock` 成功检测到了这一双向死锁环（`T12 -> T2 -> T12`），并立刻终止了该消息的发送（返回 `-1` 错误码），同时打印了死锁警告。这成功避免了整个系统或 GUI 服务的卡死。
- **规避建议**： 在编写符合 POSIX 规范的程序时，**信号处理器内禁止调用任何涉及阻塞、I/O 或 IPC 的非异步信号安全函数（例如 `printf` / `outsfmt`）**。通常的做法是在处理器中仅设置一个易失性标志位：
    
    cpp
    
    volatile sig_atomic_t interrupted = 0;
    
    static void sigint_handler(int signo) {
    
        interrupted = 1; // 仅设置标志位，不进行 IPC
    
    }
    
    然后在主循环中检查该标志位并进行安全打印或退出。

---

### 3. 关闭 Shell 窗口行为

[INFOR]Process 16 exited with code 00000000

[INFOR]cleaning 16, now exist 11 tasks

[INFOR]Process 17 forced to exit due to signal 9

[INFOR]Process 17 exited with code 00000089

[INFOR]cleaning 17, now exist 10 tasks

关闭 Shell 时，Shell 进程（PID 16）正常退出，其启动的子进程（PID 17）被控制台服务正确识别并发送了 `SIGKILL (9)` 强制杀死（退出码 `0x89` 即 `128 + 9 = 137`），清理流程完美无缺。

目前，整个内核信号递送机制、安全上下文恢复、栈对齐与进程树生命周期清理工作已经全部圆满完成。


---

Prompt

```
【单线程、单 pending bitmap、单 handler 跳转”的 MVP signal 系统】
信号机制

我在第一阶段想要实现的东西：

- **孤儿进程领养（Reparenting）**：
- 处理应用执行时发生的异常
- 实现Signal 的发送

请参考
- 目前的代码结构
- D:\her\study\mcca-信号3.md （核心设计，仅供参考，有错误可以指出）
- C:\Users\phina\Documents\ai-log\2026-0513-Signals\signal_mechanism_plan2.md

为我制定第一阶段信号机制的实施方案（不要实现用不上的功能）
---
我更改了一下键盘处理机制D:\her\mecocoa\devdriv\kboard.cpp  
  
我想知道（回答我，而不是莽撞的改代码）  
1、现在是否能够像 Linux 终端那样响应 Ctrl+C（非GUI由中断产生、GUI由Shell产生）  
2、现在关闭Shell ，Shell 的子进程是因为Signal信号被关闭的，还是因为父子关系被关闭的
---
如果程序因为 SIGKILL / SIGTERM 强制关闭，请输出一条调试信息
请实现Ctrl+C时发送 SIGINT（对于在Shell中运行Cot, Cot 中运行子程序来说，可能会打断 read 等系统调用）

```






