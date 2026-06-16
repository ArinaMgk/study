# Mecocoa 信号系统设计文档

路线：现代化 POSIX-compatible

设计POSIX兼容层，但不走 POSIX/旧Linux 的老路。

## 目录

0. [信号状态机](#0-信号状态机)
1. [架构概述](#1-架构概述)
2. [核心数据结构](#2-核心数据结构)
3. [信号处理场景](#3-信号处理场景)
4. [安全防线](#4-安全防线)
5. [系统不变量](#5-系统不变量)
6. [生命周期同步](#6-生命周期同步)
7. [架构优缺点总结](#7-架构优缺点总结)
8. [信号执行模型](#8-信号执行模型（signal-execution-model）)

---

## 0. 信号状态机

这是整个文档的地基，定义了线程在信号生命周期中的所有状态转换。

### 0.1 信号投递状态机

```
RUNNING → EXCEPTION_SUSPENDED → HANDLER_SETUP → IN_HANDLER
                                                      ↓
RUNNING → AST_ARMED → IN_TRAMPOLINE → HANDLER_SETUP  ↓
                                                  IN_SIGRETURN
                                                      ↓
                                             SIGRETURN_SUSPENDED
                                                      ↓
                                                  RESTORED → RUNNING
```

### 0.2 状态转换表

| 转换 | 触发方 | 前置不变量 | 后置不变量 |
|------|--------|-----------|-----------|
| **RUNNING → EXCEPTION_SUSPENDED** | 微内核（硬件异常） | 线程在用户态执行 | `ast_depth++`，现场压栈 |
| **EXCEPTION_SUSPENDED → HANDLER_SETUP** | POSIX Server | 异常已翻译为信号 | 栈帧已构建，RIP 指向 handler |
| **HANDLER_SETUP → IN_HANDLER** | 微内核（恢复现场） | `sys_thread_set_state_safe` 已验证 | 线程执行 handler |
| **RUNNING → AST_ARMED** | POSIX Server | `ast_pending = 1` | 线程将在 AST 检查点被劫持 |
| **AST_ARMED → IN_TRAMPOLINE** | 微内核（调度器劫持） | `ast_depth < MAX_AST_DEPTH` | RIP 指向 trampoline |
| **IN_TRAMPOLINE → HANDLER_SETUP** | 微内核（IPC 通知） | Trampoline 已执行 syscall | POSIX Server 收到通知 |
| **IN_HANDLER → IN_SIGRETURN** | 线程（调用 sigreturn） | Handler 执行完毕 | 线程执行 `sys_sigreturn` |
| **IN_SIGRETURN → SIGRETURN_SUSPENDED** | 微内核 | Cookie 校验通过 | 线程挂起，等待 POSIX Server |
| **SIGRETURN_SUSPENDED → RESTORED** | POSIX Server | 现场已清洗验证 | `ast_depth--`，现场恢复 |
| **RESTORED → RUNNING** | 微内核（恢复现场） | `ast_depth == 0` 或继续嵌套 | 线程恢复原执行流 |

### 0.3 停止状态机

```
RUNNING → STOP_ARMED → STOPPED → RUNNING
```

| 转换 | 触发方 | 前置不变量 | 后置不变量 |
|------|--------|-----------|-----------|
| **RUNNING → STOP_ARMED** | POSIX Server | `stop_pending = 1`，记录 `stop_generation` | 线程将在 AST 检查点挂起 |
| **STOP_ARMED → STOPPED** | 微内核（AST 检查点） | 线程不在敏感操作中 | 线程进入 `TASK_STOPPED` |
| **STOPPED → RUNNING** | POSIX Server（SIGCONT） | 所有停止信号 pending 已清除 | 线程恢复执行 |

### 0.4 死亡状态机

```
任意状态 → TASK_DYING → Zombie
```

| 转换 | 触发方 | 前置不变量 | 后置不变量 |
|------|--------|-----------|-----------|
| **任意 → TASK_DYING** | 微内核（SIGKILL） | Capability 校验通过 | 所有线程标记 `THREAD_FLAG_DYING` |
| **TASK_DYING → Zombie** | 微内核（SMP 安全释放） | 所有 CPU 已脱离 mm | 页表和 TCB 已释放，通知 POSIX Server |

### 0.5 执行权所有者状态机

执行权所有者模型将投递机制（AST）与控制语义（Stop/Debug）剥离，消除状态组合爆炸：

```
THREAD ──→ SIGNAL ──→ THREAD
   │           │
   │           ↓
   │      EXCEPTION
   │           │
   ↓           ↓
DEBUGGER    DEBUGGER（最高优先级）
   │
   ↓
STOPPED
   │
   ↓
THREAD（SIGCONT 恢复）
```

| 转换 | 触发方 | 前置不变量 | 后置不变量 |
|------|--------|-----------|-----------|
| **THREAD → SIGNAL** | POSIX Server | AST 投递 | `exec_owner = EXEC_OWNER_SIGNAL` |
| **SIGNAL → THREAD** | POSIX Server | sigreturn 完成 | `exec_owner = EXEC_OWNER_THREAD` |
| **THREAD → EXCEPTION** | 微内核 | 硬件异常 | `exec_owner = EXEC_OWNER_EXCEPTION` |
| **EXCEPTION → SIGNAL** | POSIX Server | 异常翻译为信号 | `exec_owner = EXEC_OWNER_SIGNAL` |
| **任意 → DEBUGGER** | 调试器 | `sys_thread_freeze` | 当前 Continuation 挂起 |
| **DEBUGGER → 原状态** | 调试器 | `sys_thread_thaw` | 恢复之前的 owner |
| **THREAD → STOPPED** | 微内核 | SIGSTOP | `exec_owner = EXEC_OWNER_STOPPED` |
| **STOPPED → THREAD** | POSIX Server | SIGCONT | `exec_owner = EXEC_OWNER_THREAD` |

**优先级规则**：
- `EXEC_OWNER_DEBUGGER` 具有最高优先级，可以直接抢占 `EXEC_OWNER_THREAD` 和 `EXEC_OWNER_SIGNAL`
- 调试器介入不走 AST 队列，直接调用微内核的 `sys_thread_freeze` 转移 Owner
- 任何 Owner 的转移都会使当前持有的 Continuation 挂起或失效

---

## 1. 架构概述

### 1.1 三层解耦架构

整个信号系统被严格切分为三个物理隔离的层级，彻底避免宏内核的面条代码。

#### 1.1.1 底层：Mecocoa 微内核 (Ring 0 - The Core)

**定位**：只提供"机制"，完全不懂 POSIX 标准，不知道什么是 SIGINT 或 SIGSEGV。

**职责**：
- 提供纯粹的 IPC 消息传递基础设施（包括异常端口）
- 提供 AST（异步系统陷阱）的调度器劫持点（仅修改现场，零 IPC）
- 提供基于 Capability Token 鉴权的线程现场篡改接口（读写内存/修改寄存器）
- 保留最高权限的 `sys_task_force_kill`

#### 1.1.2 中间层：POSIX 兼容服务 (Ring 3 - The POSIX Server)

**定位**：拥有特权的 Ring 3 用户进程集合，提供"策略"，是系统的信号大脑。

**职责**：
- 监听各级异常端口，将微内核发来的硬件异常错误码翻译为 POSIX 信号
- 维护所有进程和线程的 POSIX 状态机（`sigaction` 表、屏蔽掩码、RT 信号队列）
- 负责计算红区（Red Zone）、检查备用栈，构建 `SigStackFrame` 并通过 `sys_thread_set_state` 设置线程现场

##### 1.1.2.1 服务拆分拓扑

将单体 POSIX Server 拆分为职责单一的微服务，防止其演变为用户态宏内核（Userspace Macro-kernel）：

**Signal Server**：
- 专注信号的挂起掩码（sigmask）计算、AST 队列管理以及 RT 信号的 FIFO 调度
- 维护 `select_next_signal` 和 `select_target_thread` 的核心逻辑
- 持有 `signal_route_lock`，保证信号路由的原子性

**Process Server**：
- 接管 `waitpid` 和 Zombie 进程表
- 当线程终止时，内核直接向 Process Server 发送死亡通知
- 管理进程生命周期（fork/exec/wait/exit）

**Debug/Exception Server**：
- 专职挂载微内核的异常端口，负责断点、单步执行和同步异常的翻译
- 管理 `EXEC_OWNER_DEBUGGER` 的 Owner 转移（通过 `sys_thread_freeze`/`sys_thread_thaw`）
- 异常翻译逻辑（#PF → SIGSEGV, #GP → SIGBUS 等）

**服务间通信语义**：

在涉及进程生命周期变更的关键节点（fork、exit 等），Process Server 与 Signal Server 之间采用微内核提供的同步阻塞 IPC（Synchronous Blocking IPC）进行通信。这种强一致性设计能确保在 fork 或 exit 发生时，进程管理状态与信号路由状态的变更在同一时钟周期内对全系统可见，避免引入复杂的分布式事务回滚逻辑。对于非状态关键的高频事件（如信号查询、sigmask 读取），则通过只读共享内存映射进行无锁查询。

#### 1.1.3 顶层：用户态应用库 (Ring 3 - libc / The Trampoline)

**定位**：普通用户态进程。

**职责**：
- 提供只读映射的 `AST_TRAMPOLINE_VADDR` 页，包含纯 `syscall` 指令（不触碰栈），供线程"自首"汇报状态
- 提供标准的 `__sigreturn` 存根，负责在 Handler 执行完毕后呼叫 POSIX Server 恢复现场

#### Trampoline ABI 定义

Trampoline 可能在栈溢出、guard page 损坏、altstack 无效时被调用，**必须保证绝对不碰栈**。

```nasm
; AST_TRAMPOLINE_VADDR 的完整内容（只读共享页）
; ABI 约束：禁止 push/pop/call/ret/任何栈操作

AST_TRAMPOLINE:
    mov  rax, SYS_signal_transition   ; 系统调用号
    syscall                           ; 陷入内核，携带 tid（在 rdi）
    ud2                               ; 不可达（微内核不会从这里返回用户态）
                                      ; POSIX Server 已修改 RIP，返回落入 handler
```

**ABI 约束**：
- **零栈操作**：不使用 `push`/`pop`/`call`/`ret`，不依赖栈指针
- **只读共享页**：Trampoline 代码映射为只读，防止被篡改
- **`SYS_signal_transition`**：原子系统调用，语义是"通知内核我在 AST 检查点，挂起自己"
- **`ud2` 不可达**：微内核不会从 Trampoline 返回用户态，POSIX Server 已修改 RIP 指向 handler

---

## 2. 核心数据结构

### 2.1 状态分离模型 (运行于 POSIX Server)

```c
// 信号来源可信度模型
struct SignalOrigin {
    enum {
        SRC_KERNEL_EXCEPTION,   // 内核异常（#PF, #GP 等），最高可信度
        SRC_USER_IPC,           // 用户进程通过 kill()/pthread_kill() 发送
        SRC_DEBUGGER,           // 调试器注入
        SRC_TIMER,              // 定时器信号
        SRC_POSIX_SERVER        // POSIX Server 内部生成
    } source;
    capability_t sender_cap;    // 发送者 Capability 快照（用于审计）
    uint64_t     audit_seq;     // 审计序列号（用于日志追踪）
};

struct RTSigEntry {
    siginfo_t           info;
    uint64_t            enqueue_order;  // 入队序号（用于 FIFO 排序）
    struct SignalOrigin origin;         // 来源可信度信息
    TAILQ_ENTRY(RTSigEntry) entries;
};

struct ThreadSignalState {
    sigset_t          blocked_mask;       
    sigset_t          standard_pending;   
    
    // RT 信号队列：按入队顺序 FIFO 投递，不按编号重新排序
    // POSIX 规定：RT 信号必须按发送顺序投递，同编号的多个实例都要投递
    // Linux 行为：RT 信号按入队顺序 FIFO，标准信号 bitmap collapse
    TAILQ_HEAD(rt_queue, RTSigEntry) rt_queues[SIGRTMAX - SIGRTMIN + 1];
    uint64_t          rt_enqueue_counter;  // 全局入队计数器（用于 FIFO 排序）
};

struct ProcessSignalState {
    sigaction_t       handlers[NSIG];     
    sigset_t          process_pending;    
    bool              signal_freeze;      // Fork 窗口期冻结标志
    TAILQ_HEAD(, RTSigEntry) freeze_pending_queue; // 窗口期临时暂存队列
    
    // SIGSTOP 合作式挂起确认机制
    atomic_int        stop_pending_count; // 等待确认的线程数
    uint64_t          stop_generation;    // 停止世代号
    
    // 信号路由保护
    spinlock_t        signal_route_lock;  // 保护 process_pending → thread 路由
    int               signal_rr_cursor;   // round-robin 游标，防线程饥饿
    
    ThreadSignalState threads[];          
};
```

### 2.2 信号投递顺序策略（POSIX 兼容）

#### 2.2.1 Process-Directed 信号线程选择算法

当信号目标是进程（而非特定线程）时，需要选择合适的线程来投递。以下算法在持有 `signal_route_lock` 时执行：

```
process-directed 信号线程选择优先级：

1. 优先选 SIGNAL_WAIT 状态的线程（正在 sigwaitinfo/sigsuspend）
2. 其次选 SIGEXEC_NONE 且未屏蔽该信号的线程
3. 排除 SIGEXEC_HANDLER 中的线程（已在处理其他信号）
4. 排除 signal_defer_depth > 0 且未超时的线程
5. 以 signal_rr_cursor 做 round-robin，防饥饿
6. 若所有线程都屏蔽：信号留在 process_pending 等待解除屏蔽
```

```c
// 选择接收 process-directed 信号的线程
// 调用方必须持有 signal_route_lock
thread_t *select_target_thread(ProcessSignalState *pss, int signo) {
    // Phase 1: 优先选 SIGNAL_WAIT 状态的线程
    for_each_thread(thread, pss->process) {
        if (thread->state == TASK_SIGNAL_WAIT &&
            !sigismember(&thread->blocked_mask, signo)) {
            return thread;
        }
    }
    
    // Phase 2: 选 SIGEXEC_NONE 且未屏蔽的线程（round-robin）
    int nthreads = pss->process->thread_count;
    int start = pss->signal_rr_cursor % nthreads;
    
    for (int i = 0; i < nthreads; i++) {
        int idx = (start + i) % nthreads;
        thread_t *thread = pss->process->threads[idx];
        
        // 排除正在处理信号的线程
        if (thread->sigexec_state != SIGEXEC_NONE)
            continue;
        
        // 排除屏蔽该信号的线程
        if (sigismember(&thread->blocked_mask, signo))
            continue;
        
        // Phase 4: 排除 defer 中且未超时的线程
        if (atomic_read(&thread->user_view->signal_defer_depth) > 0) {
            uint64_t elapsed = tsc_to_ns(rdtsc() - thread->defer_enter_tsc);
            if (elapsed < MAX_DEFER_NS)
                continue;  // 未超时，跳过
        }
        
        // 找到合适线程，更新游标
        pss->signal_rr_cursor = (idx + 1) % nthreads;
        return thread;
    }
    
    // Phase 6: 所有线程都屏蔽，留在 process_pending
    return NULL;
}
```

#### 2.2.2 信号选择函数

```c
// 信号投递选择函数
// 调用方必须持有 signal_route_lock
int select_next_signal(ThreadSignalState *tss) {
    // === Phase 1: 标准信号（bitmap collapse） ===
    // 标准信号：多个相同信号只投递一次，不排队
    // POSIX 规定：标准信号要么投递，要么不投递，没有"多次"概念
    for (int signo = 1; signo < SIGRTMIN; signo++) {
        if (sigismember(&tss->standard_pending, signo) && 
            !sigismember(&tss->blocked_mask, signo)) {
            // 找到第一个未屏蔽的标准信号
            sigdelset(&tss->standard_pending, signo);
            return signo;
        }
    }
    
    // === Phase 2: RT 信号（FIFO by enqueue order） ===
    // RT 信号：按入队顺序投递，不按编号重新排序
    // POSIX 规定：RT 信号必须按发送顺序投递，同编号的多个实例都要投递
    // Linux 行为：RT 信号按入队顺序 FIFO，不按编号优先级
    
    uint64_t min_enqueue_order = UINT64_MAX;
    int selected_signo = 0;
    struct RTSigEntry *selected_entry = NULL;
    
    // 遍历所有 RT 信号队列，找到最早入队的信号
    for (int signo = SIGRTMIN; signo <= SIGRTMAX; signo++) {
        if (sigismember(&tss->blocked_mask, signo)) {
            continue;  // 跳过被屏蔽的信号
        }
        
        struct RTSigEntry *entry = TAILQ_FIRST(&tss->rt_queues[signo - SIGRTMIN]);
        if (entry && entry->enqueue_order < min_enqueue_order) {
            min_enqueue_order = entry->enqueue_order;
            selected_signo = signo;
            selected_entry = entry;
        }
    }
    
    if (selected_entry) {
        // 从队列中移除并返回
        TAILQ_REMOVE(&tss->rt_queues[selected_signo - SIGRTMIN], selected_entry, entries);
        return selected_signo;
    }
    
    return 0;  // 没有待投递的信号
}

// RT 信号入队函数
void enqueue_rt_signal(ThreadSignalState *tss, int signo, siginfo_t *info) {
    struct RTSigEntry *entry = malloc(sizeof(*entry));
    entry->info = *info;
    entry->enqueue_order = tss->rt_enqueue_counter++;  // 分配入队序号
    TAILQ_INSERT_TAIL(&tss->rt_queues[signo - SIGRTMIN], entry, entries);
}

// 标准信号入队函数（collapse）
void enqueue_standard_signal(ThreadSignalState *tss, int signo) {
    // 标准信号：直接设置 bitmap，多次发送等同于一次
    sigaddset(&tss->standard_pending, signo);
}
```

**POSIX 信号投递语义总结**：

| 信号类型 | 排队行为 | 投递顺序 | 多次发送 |
|---------|---------|---------|---------|
| 标准信号 (1-31) | 不排队（collapse） | 编号顺序 | 只投递一次 |
| RT 信号 (SIGRTMIN-SIGRTMAX) | 排队 | FIFO（入队顺序） | 每次都投递 |

**关键区别**：
- **标准信号**：`SIGUSR1` 发送 10 次，只投递 1 次
- **RT 信号**：`SIGRTMIN+1` 发送 10 次，投递 10 次，按发送顺序

**Linux 兼容性**：
- Linux 的 RT 信号投递严格按照入队顺序，不按编号优先级
- 如果先发送 `SIGRTMIN+3`，再发送 `SIGRTMIN+1`，投递顺序是 `SIGRTMIN+3` 先，`SIGRTMIN+1` 后
- 这与"最低编号优先"不同，必须按入队顺序

### 2.3 核心控制块与安全鉴权令牌 (微内核侧)

微内核调度器依赖 `ast_pending`，而跨进程修改依赖 `CapabilityToken`。

```c
#define MAX_AST_DEPTH 8  // 嵌套信号最大深度

// AST 上下文类型：区分不同的可重入来源
enum AstContext {
    AST_NONE,       // 不在 AST 中
    AST_SIGNAL,     // 普通信号处理
    AST_SIGRETURN,  // sigreturn 执行中
    AST_EXCEPTION,  // 异常处理（#PF, #GP 等）
    AST_TRAMPOLINE, // 蹦床执行中
};

// 线程信号执行状态（替换旧的 in_ast_checkpoint 布尔值）
enum ThreadSignalExecutionState {
    SIGEXEC_NONE,        // 无信号上下文，正常执行
    SIGEXEC_TRANSITION,  // 在 trampoline 或 AST 检查点，等待 POSIX Server
    SIGEXEC_HANDLER,     // 正在执行 signal handler
    SIGEXEC_SIGRETURN,   // 正在执行 __sigreturn
};

// 执行权所有者模型：将投递机制与控制语义剥离
enum ThreadExecutionOwner {
    EXEC_OWNER_THREAD,    // 正常用户态执行
    EXEC_OWNER_SIGNAL,    // 执行 AST 信号 handler
    EXEC_OWNER_DEBUGGER,  // 被外部调试器冻结
    EXEC_OWNER_STOPPED,   // 被调度器冻结（SIGSTOP）
    EXEC_OWNER_EXCEPTION, // 处理同步异常
};

struct ThreadControlBlock {
    atomic_t    ast_pending;    // 核心劫持点
    bool        stop_pending;   // SIGSTOP 合作式挂起标记
    bool        signal_deferred; // 信号延迟标记（critical section 内）
    bool        need_resched;   // 需要调度标志（IPI handler 设置）
    uint64_t    stop_generation; // 停止世代号（防止过期确认）
    int         ast_depth;      // 当前 AST 栈深度
    enum ThreadSignalExecutionState sigexec_state;  // 信号执行状态
    enum ThreadExecutionOwner exec_owner;  // 执行权所有者
    int         signal_nesting_depth;  // 当前嵌套层数（== ast_depth）
    struct ThreadUserView *user_view;  // 映射到用户态的共享视图
    uint64_t    defer_enter_tsc;       // 进入 defer 时的 TSC 时间戳（看门狗）
    struct SyscallContinuation *current_continuation; // 当前线程持有的续传令牌
    uint32_t    flags;          // 线程标志（THREAD_FLAG_DYING 等）
    
    // AST 栈：支持嵌套信号处理，每一帧是一个 SignalTransition 对象
    struct SignalTransition ast_stack[MAX_AST_DEPTH];
};

// 信号转换对象（AST 栈的每一帧）
// debugger、unwinder、语言运行时统一使用此对象
struct SignalTransition {
    HardwareInterruptFrame interrupted;  // 被打断时的完整现场
    siginfo_t              info;         // 信号信息（含 SignalOrigin）
    ucontext_t             context;      // 用户态上下文（供 debugger/unwinder）
    enum TransitionReason {
        TRANS_SIGNAL,     // 普通异步信号
        TRANS_EXCEPTION,  // 硬件异常
        TRANS_STOP,       // SIGSTOP
        TRANS_DEBUG,      // 调试器断点
    } reason;
    uint64_t cookie;                     // SROP 防御校验值
};

// 线程标志定义
#define THREAD_FLAG_DYING      (1 << 0)  // 线程正在死亡（SIGKILL shootdown）

struct CapabilityToken {
    uint64_t  opaque_id;      
    pid_t     target_pid;     
    uint32_t  permissions;    // 细粒度权限位图
    uint64_t  expiry_version; // 杜绝 Use-After-Free
};
```

### 2.3.1 SignalTarget 抽象

信号投递目标可以是线程、进程、进程组或会话：

```c
// 信号投递目标抽象
struct SignalTarget {
    enum {
        TARGET_THREAD,
        TARGET_PROCESS,
        TARGET_PGRP,
        TARGET_SESSION
    } type;
    union {
        tid_t   tid;
        pid_t   pid;
        pid_t   pgid;
        pid_t   session_id;
    };
};
```

POSIX Server 根据 `SignalTarget.type` 解析投递范围：
- `TARGET_THREAD`：精确投递到指定线程
- `TARGET_PROCESS`：投递到进程内任意可接收信号的线程
- `TARGET_PGRP`：投递到进程组内所有进程
- `TARGET_SESSION`：投递到会话内所有进程

### 2.4 Capability 细粒度权限定义

原始 `CAP_PERM_WRITE_STATE` 权限过大，拆分为细粒度权限：

```c
// --- Capability 细粒度权限定义（最小权限原则） ---
#define CAP_PERM_READ_STATE        (1 << 0)  // 允许 sys_thread_get_state
#define CAP_PERM_SIGNAL_DELIVERY   (1 << 1)  // 允许信号投递（RIP 必须指向 registered handler）
#define CAP_PERM_EXCEPTION_RECOVER (1 << 2)  // 允许异常恢复（RIP 必须指向 registered handler）
#define CAP_PERM_SIGRETURN_RESTORE (1 << 3)  // 允许 sigreturn 恢复现场（RIP 必须通过验证）
#define CAP_PERM_READ_MEM          (1 << 4)  // 允许 sys_process_read_memory（必须不破坏 COW）
#define CAP_PERM_WRITE_MEM         (1 << 5)  // 允许 sys_process_write_memory（仅限 sigframe 区域）
#define CAP_PERM_FORCE_KILL        (1 << 6)  // 允许 sys_task_force_kill
#define CAP_PERM_SUSPEND           (1 << 7)  // 允许 sys_thread_suspend/resume
```

### 2.5 RIP 验证策略

```c
// --- RIP 验证策略（防止任意代码执行） ---
// 核心原则：微内核不维护 POSIX 语义的 handler 白名单
// 只检查 Capability 权限和内存属性：RIP 是否指向目标进程的可执行用户态页面

// 验证函数：基于 Capability + 内存属性的 RIP 验证
bool validate_rip_pure_mechanism(thread_t *thread, uint64_t rip, capability_t token) {
    // 检查 1：Canonical 规范
    if (!is_canonical(rip)) {
        return false;
    }
    
    // 检查 2：用户空间范围
    if (!is_user_address(rip)) {
        return false;
    }
    
    // 检查 3：页表属性
    // 该地址必须是 PROT_EXEC、非内核页
    struct vm_area *vma = find_vma(thread->pid, rip);
    if (!vma || !(vma->flags & VM_EXEC)) {
        return false;
    }
    if (vma->flags & VM_KERNEL) {
        return false;  // 防止跳入 vDSO 内核代码
    }
    
    // 检查 4：Capability 权限
    // 验证调用者是否有权修改目标线程的执行状态
    if (!capability_check(token, CAP_PERM_MODIFY_EXEC_STATE)) {
        return false;
    }
    
    return true;  // 无需知道这是不是 sigaction 注册的地址
}

// --- sys_thread_set_state 的安全封装 ---
// 微内核提供的安全接口，强制执行 RIP 验证
long sys_thread_set_state_safe(
    tid_t tid,
    struct HardwareInterruptFrame *new_frame,
    capability_t token
) {
    thread_t *thread = lookup_thread(tid);
    
    // 强制验证 RIP（Capability + 内存属性）
    if (!validate_rip_pure_mechanism(thread, new_frame->rip, token)) {
        return -EINVAL;  // RIP 验证失败
    }
    
    // 强制清洗 RFLAGS 特权位
    new_frame->rflags = (new_frame->rflags & 0x8D5) | 0x200;
    
    // 强制重置段寄存器
    new_frame->cs = USER_CS;
    new_frame->ss = USER_SS;
    
    // 应用新状态
    *thread->current_frame = *new_frame;
    return 0;
}
```

**设计说明**：
- **微内核只检查页表**（已有的基础设施），不引入任何 POSIX 概念
- **防止 POSIX Server exploit** 将 RIP 指向内核地址或不可执行区域
- **代价**：无法防止"跳到进程内的任意合法可执行地址"，但这已经足够缩小攻击面
- **优势**：不破坏"微内核不懂 POSIX"的核心设计原则

### 2.6 sys_thread_get_state 返回结构

POSIX Server 通过此结构获取线程状态。当线程处于 AST 检查点时，附带栈顶的预劫持快照：

```c
struct ThreadState {
    HardwareInterruptFrame  current;              // 当前执行状态（可能在 Trampoline 内部）
    
    // 信号执行状态
    enum ThreadSignalExecutionState sigexec_state;  // 当前信号执行阶段
    int                     ast_depth;            // 当前 AST 栈深度
    HardwareInterruptFrame  ast_top_frame;        // 栈顶的劫持前现场
    uint64_t                ast_top_cookie;       // 栈顶的 sigreturn 校验值
};
```

POSIX Server 调用 `sys_thread_get_state(tid)` 后，根据 `state.sigexec_state` 判断：
- 若为 `SIGEXEC_TRANSITION`：使用 `state.ast_top_frame` 填写 `ucontext_t`，这才是 `sigreturn` 后要恢复的完整现场
- 若为 `SIGEXEC_NONE`：使用 `state.current` 获取当前执行状态
- 若为 `SIGEXEC_HANDLER` 或 `SIGEXEC_SIGRETURN`：线程正在执行 handler 或 sigreturn，不干预

### 2.7 投递模式分类

系统存在两条信号投递路径，必须明确其语义边界，避免互相污染。

#### 2.7.1 投递模式分类表

| 信号来源 | 投递模式 | 理由 |
|---------|---------|------|
| 硬件异常 (#PF, #GP) | **Forced，同步** | 必须立即处理，无法延迟 |
| SIGKILL/SIGSTOP | **Forced，内核直接** | Liveness 保证，绕过 POSIX Server |
| SIGSEGV（handler 内部） | **Forced，同步** | 必须立即递归或终止 |
| kill()/pthread_kill() | **Cooperative，AST** | 允许安全点延迟 |
| 定时器信号 | **Cooperative，AST** | 精度要求不高于调度粒度 |
| IPI 强制抢占 | **Forced → Cooperative 过渡** | 强制进入 kernel boundary 后走 AST 路径 |

#### 2.7.2 两条路径的语义边界

| 语义要素 | Cooperative 路径 | Forced 路径 |
|---------|-----------------|-------------|
| **sigmask 判断时机** | AST 检查点（返回用户态边界） | 异常/系统调用入口（立即） |
| **nested delivery 栈帧** | 在 AST 栈中构建 | 在异常帧上直接构建 |
| **restart_cookie 生命周期** | 完整（可跨多次投递） | 立即作废（强杀不重启） |
| **用户态协作安全点** | 尊重 `signal_defer_depth` | 忽略，强制投递 |
| **IPI 触发** | 仅当线程卡在用户态 | 不需要（已在内核态） |

#### 2.7.3 路径转换规则

```
┌─────────────────────────────────────────────────────────────────┐
│                    信号投递路径决策流程                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  信号到达                                                        │
│      │                                                          │
│      ▼                                                          │
│  ┌─────────────────┐                                            │
│  │ 是否为 Forced？ │                                            │
│  │ (异常/SIGKILL)  │                                            │
│  └────────┬────────┘                                            │
│           │                                                     │
│     ┌─────┴─────┐                                               │
│     │           │                                               │
│     ▼           ▼                                               │
│   Yes          No                                               │
│     │           │                                               │
│     ▼           ▼                                               │
│ ┌─────────┐  ┌─────────────────┐                                │
│ │ Forced  │  │ Cooperative     │                                │
│ │ 同步处理│  │ 设置 ast_pending│                                │
│ └─────────┘  └────────┬────────┘                                │
│                       │                                         │
│                       ▼                                         │
│               ┌─────────────────┐                               │
│               │ 线程是否在       │                               │
│               │ 用户态运行？     │                               │
│               └────────┬────────┘                               │
│                        │                                        │
│                  ┌─────┴─────┐                                  │
│                  │           │                                  │
│                  ▼           ▼                                  │
│                Yes          No                                  │
│                  │           │                                  │
│                  ▼           ▼                                  │
│           ┌──────────┐  ┌──────────┐                            │
│           │ 发送 IPI │  │ 等待 AST │                            │
│           │ 强制抢占 │  │ 检查点   │                            │
│           └────┬─────┘  └──────────┘                            │
│                │                                                 │
│                ▼                                                 │
│         ┌──────────────┐                                         │
│         │ IPI 触发后   │                                         │
│         │ 进入 AST 路径│                                         │
│         └──────────────┘                                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**关键设计决策**：
- **IPI 不是独立路径**：IPI 的作用是强制线程进入 kernel boundary，之后仍走 AST cooperative 路径
- **Forced 路径极简**：只有硬件异常和 SIGKILL/SIGSTOP 走 forced 路径，其他一律走 cooperative
- **语义一致性**：一旦进入 AST 路径，无论是自愿还是 IPI 强制，后续处理完全相同

### 2.8 信号投递边界（Signal Delivery Boundary）

系统只在以下确定性边界投递信号，**不在任意指令处注入**：

| 边界 | 是否允许投递 | 说明 |
|------|------------|------|
| syscall 返回用户态 | YES | 最常见路径 |
| 中断处理返回用户态 | YES | Timer/IPI 触发后 |
| 异常处理返回用户态 | YES | 经过 AST 统一路径 |
| 显式 `sys_signal_checkpoint` | YES | 用户主动请求 |
| `signal_defer_depth` 归零 | YES | 离开 critical section |
| 任意用户态指令中间 | NO | 永远不会发生 |
| `signal_defer_depth > 0` 时 | NO（受限） | 见 defer 预算规则 |

这是系统与 libc/runtime/JIT 编译器的合约（ABI）。

**核心语义**：
- 任何组件依赖"信号只在边界投递"这一语义，而非"随机打断"
- libc 的 critical section 通过 `signal_defer_depth` 控制投递时机
- JIT 编译器无需在生成的代码中插入信号检查点
- runtime 可以在确定的安全点执行 GC、锁操作等敏感逻辑

---

## 3. 信号处理场景

### 3.1 场景 A：硬件异常处理（统一 AST 路径）

当线程触发硬件异常（如 #PF、#GP）时，**异常也必须先压入 AST 栈，走与异步信号完全相同的路径**：

```
#PF 触发
→ 内核异常入口（Ring 0）
→ ast_push(AST_EXCEPTION, 当前帧)           ← 统一 AST 路径
→ 线程状态切换为 EXCEPTION_SUSPENDED
→ 通过异常端口通知 POSIX Server
→ POSIX Server 翻译为 POSIX 信号（#PF → SIGSEGV）
→ POSIX Server 构建普通 sigframe（与异步信号完全相同的格式）
→ sys_thread_set_state_safe
→ 线程落入 handler
→ sigreturn（与异步信号完全相同的路径）
```

**关键设计决策**：
- **只有一种 sigframe 格式**：硬件异常和异步信号使用完全相同的栈帧结构
- **只有一种 sigreturn 路径**：异常恢复和信号恢复走相同的 `__sigreturn` 流程
- **异常即信号**：硬件异常通过 AST 栈统一建模，消除两套语义的分裂

**处理流程**：

1. **微内核捕获异常**：硬件异常触发，CPU 陷入 Ring 0
2. **AST 压栈**：微内核调用 `ast_push(AST_EXCEPTION, 当前帧)`，将异常现场压入 AST 栈
3. **状态切换**：线程状态切换为 `EXCEPTION_SUSPENDED`
4. **异常端口转发**：微内核通过异常端口向 POSIX Server 发送 IPC 通知
5. **信号翻译**：POSIX Server 根据异常类型翻译为对应的 POSIX 信号（如 #PF → SIGSEGV）
6. **Handler 投递**：POSIX Server 构建信号栈帧，通过 `sys_thread_set_state_safe` 设置线程执行流
7. **Handler 执行**：线程返回用户态执行信号 Handler
8. **sigreturn 恢复**：Handler 执行完毕，调用 `__sigreturn` 返回微内核，POSIX Server 恢复原执行流

### 3.2 场景 A++：SIGKILL/SIGSTOP 内核原生处理

`SIGKILL` 和 `SIGSTOP` 由微内核 Ring 0 直接处理，不依赖 POSIX Server，确保系统"活性（Liveness）"。

#### 3.2.1 libc 拦截路径

libc 对 `SIGKILL`/`SIGSTOP` 走独立的直达微内核路径，不经过 POSIX Server：

```c
// libc 的 kill() 实现
int kill(pid_t pid, int sig) {
    if (sig == SIGKILL || sig == SIGSTOP) {
        // 直接发往微内核专用端口，不经过 POSIX Server
        return sys_privileged_signal(pid, sig, my_capability_token);
    }
    return ipc_send(POSIX_SERVER_PORT, MSG_KILL, pid, sig);
}
```

#### 3.2.2 处理流程

1. **直达微内核**：`kill(pid, SIGKILL)` 通过 `sys_privileged_signal` 直达微内核
2. **Capability 校验**：微内核校验发送者的 Capability Token
3. **SMP 安全释放**：
   - 标记所有线程为 `TASK_DYING`
   - 获取所有 CPU 的 runqueue lock
   - 发送 IPI shootdown 到相关 CPU
   - 等待所有 CPU 确认已脱离进程地址空间
   - 安全释放页表和 TCB
4. **死亡通知**：微内核向 POSIX Server 发送异步死亡通知（仅做 Bookkeeping）

**设计说明**：
- **方案选择**：采用 libc 直达微内核路径，而非 POSIX Server 转发
- **优势**：即使 POSIX Server 死锁、OOM 或消息队列被塞满，管理员仍能杀掉恶意进程
- **SIGSTOP 同理**：同样走直达微内核路径，确保系统可控

### 3.3 场景 B：异步软信号投递

当 POSIX Server 决定向线程投递异步信号时：

1. **设置 AST 标志**：POSIX Server 调用 `sys_thread_set_ast_pending(tid, true)`
2. **IPI 强制抢占**：若线程正在其他 CPU 上运行，微内核发送 IPI 强制打断
3. **AST 检查点**：线程在返回用户态边界时检查 `ast_pending`
4. **调度器劫持**：微内核保存当前现场到 AST 栈，修改 RIP 指向 Trampoline
5. **自首 IPC**：线程执行 Trampoline 中的 `syscall`，向 POSIX Server 汇报
6. **Handler 设置**：POSIX Server 构建信号栈帧，设置 Handler 执行流

### 3.4 场景 B+：用户态协作安全点

解决 async-signal-safe 问题，引入用户态主动声明的安全点。

#### 3.4.1 ThreadUserView 共享内存映射

`ThreadControlBlock` 定义在 Ring 0，用户态进程无法直接访问。`signal_defer_depth` 和 `ast_pending` 的访问需要一个共享内存映射，类似 Linux 的 VVAR 页：

```c
// 每个线程创建时，微内核映射一个只读（部分字段可写）的共享页
// 映射到固定的用户态地址（可通过 TLS 找到）
struct ThreadUserView {
    // 用户态可读/写字段（通过共享页直接访问，零 syscall 开销）
    atomic_int  signal_defer_depth;  // 用户态可写：协作安全点计数
    atomic_int  ast_pending;         // 用户态只读：信号待投递标志
    
    // 填充到 cache line 对齐，防止 false sharing
    uint8_t     _pad[56];
};

// 微内核在 TCB 中保存指向同一物理页的内核视图指针
struct ThreadControlBlock {
    struct ThreadUserView *user_view;  // 映射到用户态的共享视图
    // ...内核私有字段不在共享页中
};
```

用户态代码通过固定地址（或 TLS 寄存器偏移）直接访问 `ThreadUserView`，无需 syscall，性能开销与直接读写内存相同。

#### 3.4.1.1 内存屏障与 SMP 语义

在支持 ARM 和 RISC-V 等弱内存序（Weakly Ordered）架构时，`signal_defer_depth` 与 `ast_pending` 的交互必须建立严格的 Happens-Before 关系，防止 Store-Load 重排。

```c
void enter_critical_section(ThreadUserView *view) {
    // Acquire semantics: 后续内存访问不能重排到此之前
    atomic_fetch_add_explicit(&view->signal_defer_depth, 1,
                              memory_order_acquire);
}

void leave_critical_section(ThreadUserView *view) {
    // Release semantics: 之前的内存访问不能重排到此之后
    atomic_fetch_sub_explicit(&view->signal_defer_depth, 1,
                              memory_order_release);
    
    // Full memory barrier: ARM/RISC-V 必须，防止 Store-Load 重排
    atomic_thread_fence(memory_order_seq_cst);
    
    if (atomic_load_explicit(&view->signal_defer_depth,
                             memory_order_relaxed) == 0) {
        if (atomic_load_explicit(&view->ast_pending,
                                 memory_order_acquire) != 0) {
            sys_signal_checkpoint();
        }
    }
}
```

**内核侧 IPI 触发语义**：
- 内核设置 `ast_pending` 时必须使用 `memory_order_release`
- 随后触发 IPI，IPI 的硬件中断投递在目标 CPU 上隐式构成 Acquire 语义
- 两者共同完成跨核同步的 Happens-Before 链

#### 3.4.2 用户态运行时库使用示例

```c
// 用户态运行时库使用示例
// 通过 TLS 获取 ThreadUserView 指针
static inline struct ThreadUserView *get_thread_user_view(void) {
    return (struct ThreadUserView *)tls_get(TLS_THREAD_USER_VIEW);
}

void malloc_critical_section() {
    struct ThreadUserView *view = get_thread_user_view();
    
    // 进入 critical section：延迟信号
    atomic_inc(&view->signal_defer_depth);
    
    // ... 执行 malloc 等敏感操作 ...
    
    // 离开 critical section：检查安全点
    atomic_dec(&view->signal_defer_depth);
    if (atomic_read(&view->signal_defer_depth) == 0 && 
        atomic_read(&view->ast_pending)) {
        sys_signal_checkpoint();  // 主动触发信号投递
    }
}
```

**机制**：
- `signal_defer_depth > 0`：信号延迟，不在 AST 边界投递
- `signal_defer_depth == 0`：允许信号投递
- `sys_signal_checkpoint`：用户态主动请求信号检查

#### 3.4.3 Defer 看门狗机制

**问题**：用户可以写 `signal_defer_depth++; while(1){}` 导致 SIGSTOP、调试器永远无法介入。

**解法**：TCB 里加入时间戳，调度器检查超时：

```c
// 进入 defer 时记录时间
void enter_signal_defer(thread_t *t) {
    if (atomic_inc_return(&t->user_view->signal_defer_depth) == 1)
        t->defer_enter_tsc = rdtsc();  // 第一次 defer 时记录
}

// 调度器出口处检查（与 AST 检查放在一起）
void scheduler_check_ast(thread_t *t, HardwareInterruptFrame *frame) {
    if (atomic_read(&t->user_view->signal_defer_depth) > 0) {
        uint64_t elapsed_ns = tsc_to_ns(rdtsc() - t->defer_enter_tsc);
        if (elapsed_ns > MAX_DEFER_NS) {
            // defer 超时：强制某些信号穿透
            if (t->pending_sigstop || t->pending_debug_stop)
                force_cooperative_stop(t);  // SIGSTOP/调试器不允许无限延迟
            // SIGKILL 永远不受 defer 控制（走内核直达路径）
        }
        return;  // 未超时：正常延迟
    }
    // ...正常 AST 检查
}
```

#### 3.4.4 Defer 预算规则

不同信号的 defer 上限不同，而非一刀切：

| 信号类型 | defer 策略 |
|---------|-----------|
| SIGKILL | 永不 defer（内核直达，已实现） |
| SIGSTOP / 调试器停止 | defer 有上限（防止调试器永远进不来） |
| SIGINT/SIGTERM/SIGUSR | 真正可无限 defer（用户业务逻辑的锁） |
| 定时器信号 | 可无限 defer（用户控制） |

**ABI 约束**：
- `MAX_DEFER_NS` 仅约束 SIGSTOP/调试器信号，不影响普通业务信号
- GC stop-the-world、大锁、长时间原子操作可以安全使用 defer，不会被强制打断
- SIGKILL 永远穿透，确保系统活性（Liveness）

### 3.5 场景 C：系统调用重启

当阻塞式系统调用被信号打断时：

1. **信号打断**：系统调用执行期间，信号到达
2. **重启判断**：若 Handler 设置了 `SA_RESTART`，且系统调用支持重启
3. **Cookie 记录**：微内核生成 `restart_cookie`，记录在 TCB 的 `current_restart_cookie`
4. **驱动状态保存**：驱动保存已完成字节数等状态
5. **Handler 执行**：信号 Handler 执行
6. **重启恢复**：
   - 若 Handler 正常返回：调用 `sys_restart_syscall`，驱动使用 Cookie 恢复
   - 若 Handler 调用 `siglongjmp`：libposix 主动调用 `sys_abandon_continuation()` 清零所有 cookie
   - 若后续执行新系统调用：自动清零 Cookie

#### 3.5.1 续传令牌失效场景

`restart_cookie` 在以下场景必须失效：

| 场景 | 失效机制 | 责任方 |
|------|---------|--------|
| **Handler 正常返回** | `sys_restart_syscall` 消费 cookie 后清零 | 微内核 |
| **siglongjmp** | libposix 主动调用 `sys_abandon_continuation()` | libposix |
| **新系统调用** | 任何新 syscall 入口自动清零 | 微内核 |
| **线程退出** | TCB 销毁时自动清零 | 微内核 |

**siglongjmp 特殊处理**：

```c
// libposix 的 siglongjmp 实现
void siglongjmp(sigjmp_buf env, int val) {
    // 通知内核放弃所有待续传对象（续传令牌清零）
    sys_abandon_continuation();
    // Unwind microkernel AST stack and reset exec_owner
    sys_ast_unwind(env->saved_ast_depth);
    longjmp((void*)env, val);
}
```

`siglongjmp` 可以在 handler 里不执行任何 syscall 直接跳走，因此必须在 libposix 层主动通知内核清零所有 cookie，防止续传令牌泄漏。

**AST 栈回溯语义**：当执行流越过正常的 `__sigreturn` 直接跳转时，`sys_ast_unwind` 系统调用负责将目标线程的 `ast_depth` 恢复至 `sigsetjmp` 保存的深度，并同步更新其 `exec_owner`。若回溯后 `ast_depth` 为零，内核需将状态切换回 `EXEC_OWNER_THREAD`。

```c
// 内核侧 sys_ast_unwind 实现
long sys_ast_unwind(thread_t *thread, int target_ast_depth) {
    if (target_ast_depth < 0 || target_ast_depth > thread->ast_depth)
        return -EINVAL;
    
    // 逐帧弹出 AST 栈至目标深度
    while (thread->ast_depth > target_ast_depth) {
        thread->ast_depth--;
        // 释放 SignalTransition 资源
    }
    thread->signal_nesting_depth = thread->ast_depth;
    
    // 同步更新执行权状态
    if (thread->ast_depth == 0) {
        thread->sigexec_state = SIGEXEC_NONE;
        thread->exec_owner = EXEC_OWNER_THREAD;
    } else {
        thread->sigexec_state = SIGEXEC_HANDLER;
        thread->exec_owner = EXEC_OWNER_SIGNAL;
    }
    
    return 0;
}
```

**sigsetjmp 需保存 ast_depth**：`sigjmp_buf` 必须扩展，在 `sigsetjmp` 时保存当前线程的 `ast_depth`，供 `siglongjmp` 时回溯使用。

### 3.6 场景 E：sigreturn 现场恢复闭环

当信号 Handler 执行完毕，调用 `__sigreturn` 时：

1. **sigreturn 陷入**：线程执行 `syscall` 调用 `sys_sigreturn`
2. **RSP 转发**：微内核读取陷入时的 RSP，通过 IPC 转发给 POSIX Server
3. **帧定位**：POSIX Server 根据 RSP 定位 `SigStackFrame` 地址
4. **帧读取**：通过 `sys_process_read_memory` 读取帧内容
5. **Cookie 校验**：验证帧中的 Cookie 与 AST 栈顶一致
6. **现场清洗**：
   - RIP Canonical 规范检查
   - RFLAGS 特权位清洗
   - 段寄存器重置
   - XSAVE 特性掩码清洗
7. **AST 出栈**：从 AST 栈弹出，恢复原执行流
8. **线程恢复**：通过 `sys_thread_set_state_safe` 设置恢复后的现场

**帧布局约定**：

```
Handler 入口时的栈顶 (RSP):
[8 bytes] __sigreturn 地址  ← RSP 指向这里（作为返回地址）
[N bytes] SigStackFrame     ← RSP + 8 开始

sigreturn 时 RSP = 原 RSP + 8 (ret 弹出了返回地址)
所以：SigStackFrame 地址 = sigreturn 时的 RSP
```

### 3.7 场景 F：SIGSTOP/SIGCONT 处理

#### 3.7.1 SIGSTOP 合作式挂起

采用合作式挂起，避免硬冻结导致的死锁：

1. **设置挂起标记**：POSIX Server 设置 `stop_pending = true`，记录 `stop_generation`
2. **AST 检查点**：线程在返回用户态边界检查 `stop_pending`
3. **安全挂起**：微内核将线程放入 `TASK_STOPPED` 队列
4. **确认机制**：微内核向 POSIX Server 发送 `MSG_THREAD_STOPPED` 确认
5. **世代号验证**：POSIX Server 验证 `stop_generation` 匹配，防止过期确认
6. **进程停止**：所有线程确认后，设置进程状态为 `PROC_STOPPED`

#### 3.7.2 SIGCONT POSIX 强制语义

```c
case SIGCONT:
    // 规则 1：清除所有停止信号的 pending（即使已被屏蔽）
    for each thread in process:
        clear_pending(thread, SIGSTOP | SIGTSTP | SIGTTIN | SIGTTOU);
    clear_process_pending(process, SIGSTOP | SIGTSTP | SIGTTIN | SIGTTOU);
    
    if (proc->state == PROC_STOPPED) {
        // 规则 2：即使 SIGCONT 被屏蔽，也必须唤醒已停止的进程
        for each thread: sys_thread_resume(thread->tid);
        proc->state = PROC_RUNNING;
        
        // 恢复后，若有注册的 handler，再走正常投递路径
        if (get_handler(proc, SIGCONT) != SIG_DFL && != SIG_IGN)
            deliver_signal_normally(proc, SIGCONT);
    }
    
    // SA_NOCLDSTOP 检查同 SIGSTOP
    parent_sigchld_action = get_handler(proc->ppid, SIGCHLD);
    if (!(parent_sigchld_action.sa_flags & SA_NOCLDSTOP)) {
        deliver_sigchld_to_parent(proc->ppid, CLD_CONTINUED);
    }
    break;
```

**POSIX 强制语义**：
- **规则 1**：`SIGCONT` 到达时，必须清除该进程所有待投递的停止信号（`SIGSTOP`/`SIGTSTP`/`SIGTTIN`/`SIGTTOU` 的 pending 位），即使这些信号已被屏蔽
- **规则 2**：`SIGCONT` 本身即使被屏蔽，也必须唤醒已停止的进程。若注册了 `SIGCONT` Handler，恢复后再投递

---

## 4. 安全防线

### 4.1 防线 1：SIGKILL 内核原生保底与严格 SMP 同步

`SIGKILL` 由微内核 Ring 0 直接拦截执行，不依赖 POSIX Server。即使 POSIX Server 死锁、OOM 或消息队列被塞满，管理员仍能杀掉恶意进程，确保系统"活性（Liveness）"。

**SMP 安全机制**：
- 获取所有 CPU 的 runqueue lock 后再检查线程状态和发送 IPI
- 防止线程在检查期间 migrate/preempt 导致漏发 IPI
- 确保所有 CPU 都脱离进程地址空间后才释放资源
- 防止 SMP CR3/TLB Tear-down Race 导致的三重故障

### 4.2 防线 2：SIGSTOP 合作式挂起与 SMP 竞态防护

`SIGSTOP` 采用合作式挂起机制，在 AST 边界安全挂起，避免硬冻结导致的死锁。

**SMP 竞态防护**：
- 引入确认握手机制与世代号
- 确保所有线程真正停止后才宣布进程已停止
- 防止 SMP 逃逸竞态

### 4.3 防线 3：POSIX Server 自身免疫与最小权限原则

**异常隔离**：调用 `sys_register_posix_server`。其发生的异常不走常规路由，直接上报 Init 触发灾难恢复。

**Capability 细粒度拆分**：原始的 `CAP_PERM_WRITE_STATE` 权限过大（等价于 ptrace、arbitrary RIP write），拆分为 `CAP_PERM_SIGNAL_DELIVERY`、`CAP_PERM_EXCEPTION_RECOVER`、`CAP_PERM_SIGRETURN_RESTORE` 三个细粒度权限。

**RIP 纯机制验证**：`sys_thread_set_state_safe` 强制验证 RIP 必须指向用户态可执行页面（Capability + 内存属性校验），不依赖 POSIX 语义的 handler 白名单。即使 POSIX Server 被 exploit，也无法注入任意 RIP 执行任意代码。

**防止全系统沦陷**：通过最小权限原则和 RIP 验证，确保 POSIX Server exploit 不会导致全系统 Ring 3 进程沦陷。

### 4.4 防线 4：嵌套信号防护与可重入来源区分

TCB 中维护固定深度（`MAX_AST_DEPTH = 8`）的 AST 栈，支持嵌套信号处理。每个栈帧记录 `AstContext` 类型（SIGNAL/SIGRETURN/EXCEPTION/TRAMPOLINE）和 `signo`，区分不同的可重入来源。

**递归检测**：
- 检测 SIGSEGV 递归
- 检测异常发生在 sigreturn/trampoline 中等不可恢复状态
- 立即强杀进程，防止无限递归

### 4.5 防线 5：SROP 攻击防御与现场验证

`SigStackFrame` 包含随机 Cookie 校验值，`sigreturn` 时校验与 AST 栈顶一致性。

**现场验证**：
- 恢复现场前通过 `sys_thread_set_state_safe` 的 `RIP_POLICY_SIGFRAME` 验证 RIP Canonical 规范和用户空间范围
- 清洗 XSAVE 特性掩码
- 恢复时强制清洗 RFLAGS 特权位和段寄存器
- 防止伪造栈帧提权

### 4.6 防线 6：备用栈溢出防护与红区保护

**备用栈判断与溢出检查**：若 Handler 设置了 `SA_ONSTACK`，且线程有有效的备用栈，则使用备用栈。计算 `new_rsp` 时检查是否超出备用栈底部，溢出则放弃投递并强杀进程：

```c
if ((handler_flags & SA_ONSTACK) && 
    (thread->altstack.ss_size > 0) && 
    !(thread->altstack.ss_flags & SS_DISABLE)) {
    new_rsp = thread->altstack.ss_sp + thread->altstack.ss_size;
    
    // 备用栈溢出检查
    if (new_rsp - sizeof(SigStackFrame) - 8 < thread->altstack.ss_sp) {
        // 备用栈溢出！放弃投递，直接强杀进程
        sys_task_force_kill(pid, token);
        return;
    }
}
```

**红区扣除**：无备用栈时强制扣除红区：`new_rsp = (current_rsp - 128 - sizeof(SigStackFrame) - 8) & ~0xFULL;`

**内存布局**：`new_rsp` 处放 `__sigreturn` 地址（8 字节），`new_rsp + 8` 处放 `SigStackFrame`。

### 4.7 防线 7：SIGCONT POSIX 强制语义

`SIGCONT` 到达时清除所有停止信号 pending，即使被屏蔽也必须唤醒已停止进程。发送 `SIGCHLD` 前检查父进程 `SA_NOCLDSTOP` 标志。

### 4.8 防线 8：续传令牌生命周期强绑定

TCB 中记录 `current_restart_cookie`，任何新的系统调用（非 `sys_restart_syscall`）自动清零旧 Cookie。

**防止混乱代理漏洞**：
- 防止 `siglongjmp` 逃逸导致的混乱代理漏洞
- 避免旧 Cookie 被复用 fd 的恶意程序利用

### 4.9 防线 9：用户态协作安全点

引入 `signal_defer_depth` 计数器和 `sys_signal_checkpoint` 系统调用，让运行时库（malloc、mutex、GC）主动声明 critical section。

**机制**：
- 信号只在用户态声明的安全点注入
- 真正解决 async-signal-safe 问题
- 避免用户态锁重入死锁

### 4.10 防线 10：强制进入 Kernel Boundary 的多层机制

采用多层机制，确保卡在用户态死循环、TSX transaction、用户态中断屏蔽等场景下的线程最终进入 kernel boundary：

- IPI 强制抢占（微秒级）
- Timer Tick 检查（毫秒级兜底）
- irq_work 延迟工作队列
- TSX Abort 处理
- Breakpoint 注入

解决"不可达线程"问题。

### 4.11 防线 11：sys_process_read_memory 的 COW 安全约束

Core Dump 服务读取目标进程内存时，必须保证不破坏 Copy-on-Write 语义。

**安全读取机制**：
- 使用 `get_user_pages_safe` 带 `GUP_FLAGS_READ_ONLY | GUP_FLAGS_NO_COW | GUP_FLAGS_NO_FAULT` 标志
- 确保只读访问、不触发 COW、不触发缺页
- 对于未映射的页填充 0，而不是分配新页
- 读取操作不修改目标进程的任何页表状态
- 确保 dump 本身不改变程序行为

```c
// 正确的实现（不破坏 COW）：
long sys_process_read_memory(pid_t pid, void *buf, void *target_addr, size_t len) {
    process_t *proc = lookup_process(pid);
    
    // 必须使用类似 Linux get_user_pages 的机制，带特殊标志
    struct page **pages;
    int nr_pages = get_user_pages_safe(proc, target_addr, len, &pages,
                                       GUP_FLAGS_READ_ONLY |  // 只读访问
                                       GUP_FLAGS_NO_COW);     // 不触发 COW
    
    if (nr_pages < 0) {
        return nr_pages;  // 读取失败（可能是未映射或权限问题）
    }
    
    // 直接从物理页读取，不修改页表状态
    for (int i = 0; i < nr_pages; i++) {
        void *kaddr = kmap_atomic(pages[i]);
        size_t offset = (i == 0) ? (uint64_t)target_addr % PAGE_SIZE : 0;
        size_t copy_len = min(PAGE_SIZE - offset, len);
        
        copy_to_user(buf, kaddr + offset, copy_len);
        
        buf += copy_len;
        len -= copy_len;
        
        kunmap_atomic(kaddr);
        put_page(pages[i]);  // 释放页引用
    }
    
    return 0;
}
```

**注意**：代码已经申明 no-red-zone，且任务切换时会 FXSAVE (XSAVE) FXSTORE。

---

## 5. 系统不变量

系统不变量是设计中必须始终保持成立的条件，对于正确性和安全性至关重要。任何代码路径都不得违反这些不变量。

### 5.1 AST 不变量 (AST Invariant)

```c
// 不变量：AST 栈深度与信号执行状态的一致性
// 任何时刻：
//     ast_depth > 0  ⇔  sigexec_state != SIGEXEC_NONE
//     ast_depth == 0 ⇔  sigexec_state == SIGEXEC_NONE
//     sigexec_state == SIGEXEC_TRANSITION  ⇒  线程已挂起等待 POSIX Server
//     sigexec_state == SIGEXEC_HANDLER     ⇒  线程在用户态执行 handler
//     sigexec_state == SIGEXEC_SIGRETURN   ⇒  线程正在执行 __sigreturn

// 验证函数
bool validate_ast_invariant(thread_t *thread) {
    if (thread->ast_depth > 0 && thread->sigexec_state == SIGEXEC_NONE) {
        panic("AST invariant violated: ast_depth > 0 but sigexec_state == SIGEXEC_NONE");
        return false;
    }
    if (thread->ast_depth == 0 && thread->sigexec_state != SIGEXEC_NONE) {
        panic("AST invariant violated: ast_depth == 0 but sigexec_state != SIGEXEC_NONE");
        return false;
    }
    return true;
}

// 压栈时必须同步更新执行状态
void ast_push(thread_t *thread, HardwareInterruptFrame *frame, enum TransitionReason reason, siginfo_t *info) {
    assert(thread->ast_depth < MAX_AST_DEPTH);
    
    struct SignalTransition *trans = &thread->ast_stack[thread->ast_depth];
    trans->interrupted = *frame;
    trans->info = *info;
    trans->reason = reason;
    trans->cookie = generate_sigreturn_cookie();
    
    thread->ast_depth++;
    thread->signal_nesting_depth = thread->ast_depth;
    thread->sigexec_state = SIGEXEC_TRANSITION;  // 挂起，等待 POSIX Server
}

// 出栈时根据剩余深度决定状态
void ast_pop(thread_t *thread) {
    assert(thread->ast_depth > 0);
    thread->ast_depth--;
    thread->signal_nesting_depth = thread->ast_depth;
    
    if (thread->ast_depth == 0) {
        thread->sigexec_state = SIGEXEC_NONE;  // 全部出栈，恢复正常执行
    } else {
        // 还有嵌套层，恢复到上一层的状态
        thread->sigexec_state = SIGEXEC_HANDLER;  // 继续执行外层 handler
    }
}

// 状态转换辅助函数
void transition_to_handler(thread_t *thread) {
    thread->sigexec_state = SIGEXEC_HANDLER;
}

void transition_to_sigreturn(thread_t *thread) {
    thread->sigexec_state = SIGEXEC_SIGRETURN;
}
```

### 5.2 Signal Delivery 不变量 (Signal Delivery Invariant)

```c
// 不变量 1：嵌套深度上界
//     ast_depth <= MAX_AST_DEPTH（投递前已检查，超过则强杀）
//
// 不变量 2：栈帧一致性
//     forall i in [0, ast_depth): ast_stack[i].pre_frame 有效
//     且 ast_stack[i].context 与实际来源匹配
//
// 不变量 3：异常递归检测
//     AST 栈中最多只能有一个 AST_EXCEPTION 帧
//     （异常 handler 内又触发异常是不可恢复状态）

// 验证函数
bool validate_signal_delivery_invariant(thread_t *thread) {
    // 检查 1：深度上界
    if (thread->ast_depth > MAX_AST_DEPTH) {
        panic("Signal delivery invariant violated: ast_depth exceeds MAX_AST_DEPTH");
        return false;
    }
    
    // 检查 2：异常递归（异常 handler 内又触发异常，不可恢复）
    int exception_count = 0;
    for (int i = 0; i < thread->ast_depth; i++) {
        if (thread->ast_stack[i].context == AST_EXCEPTION) {
            exception_count++;
            if (exception_count >= 2) {
                panic("Signal delivery invariant violated: nested exceptions detected");
                return false;
            }
        }
    }
    
    return true;
}

// 投递信号前检查
void deliver_signal(thread_t *thread, int signo) {
    // 确保不会超过最大嵌套深度
    if (thread->ast_depth >= MAX_AST_DEPTH) {
        // 超过最大深度，强杀进程
        sys_task_force_kill(thread->pid, SIGSEGV);
        return;
    }
    
    // 继续投递...
}
```

### 5.3 Restart 不变量 (Restart Invariant)

```c
// 内核空间对象：续传令牌（替代裸 uint64_t cookie）
struct SyscallContinuation {
    refcount_t  ref;
    tid_t       owner_tid;
    uint64_t    task_epoch;     // 防 fork 后复用
    uint64_t    object_gen;     // 防 kernel object 重用
    ssize_t     (*resume)(struct SyscallContinuation *, ssize_t done);
    void        *driver_state;
};

// 用户态句柄：代际隔离，防止 UAF
// 续传令牌直接持有驱动状态指针容易引发 Use-After-Free
// 通过全局句柄和代际号进行隔离
struct SyscallContinuationHandle {
    uint64_t    continuation_id;    // 全局唯一 ID
    uint64_t    object_generation;  // 代际号
    capability_t driver_handle;     // 驱动 Capability 句柄
};

// POSIX Server 和 libc 持有 handle，不持有内核指针
typedef struct SyscallContinuationHandle continuation_handle_t;

// 不变量：续传令牌的生命周期
//     continuation_handle 有效  ⇒  线程尚未执行新的 syscall
//     线程执行新的 syscall（非 sys_restart_syscall）⇒  handle 失效

// 验证函数
bool validate_restart_invariant(thread_t *thread) {
    // 如果有有效的 continuation handle，线程应该处于可重启状态
    if (thread->current_continuation != NULL) {
        // 检查线程是否在等待重启
        if (thread->state != TASK_INTERRUPTIBLE && 
            thread->state != TASK_RUNNING) {
            panic("Restart invariant violated: continuation exists but thread not in restartable state");
            return false;
        }
    }
    return true;
}

// 系统调用入口：清除旧 continuation
long syscall_entry_handler(thread_t *thread, long syscall_nr) {
    // 任何新的系统调用（非 sys_restart_syscall）都清除旧 continuation
    if (syscall_nr != SYS_restart_syscall) {
        if (thread->current_continuation) {
            continuation_release(thread->current_continuation);
            thread->current_continuation = NULL;
        }
    }
    
    // 继续处理系统调用...
}

// 设置续传令牌
continuation_handle_t set_continuation(
    thread_t *thread,
    ssize_t (*resume_fn)(struct SyscallContinuation *, ssize_t),
    void *driver_state
) {
    struct SyscallContinuation *cont = continuation_alloc();
    cont->owner_tid = thread->tid;
    cont->task_epoch = thread->task->epoch;
    cont->object_gen = get_object_generation();
    cont->resume = resume_fn;
    cont->driver_state = driver_state;
    
    thread->current_continuation = cont;
    return continuation_to_handle(cont);
}

// 系统调用恢复：通过代际号隔离验证
// 内核通过 driver_handle 和 object_generation 重新向驱动层发起 Lookup
// 如果驱动对象已被销毁或代际号变更，Lookup 失败并返回 EINTR
ssize_t sys_restart_syscall(thread_t *thread, continuation_handle_t handle) {
    struct SyscallContinuation *cont = thread->current_continuation;
    
    if (!cont) {
        return -EINTR;  // 无有效续传令牌
    }
    
    // 代际号校验：驱动对象是否已被销毁或替换
    if (handle.object_generation != cont->object_gen) {
        continuation_release(cont);
        thread->current_continuation = NULL;
        return -EINTR;  // 代际号不匹配，切断悬空指针
    }
    
    // Capability 校验：调用者是否有权恢复此续传
    if (!capability_check(handle.driver_handle, CAP_PERM_RESTART_SYSCALL)) {
        continuation_release(cont);
        thread->current_continuation = NULL;
        return -EINTR;
    }
    
    // 校验通过，执行恢复
    return cont->resume(cont, 0);
}

// siglongjmp 场景：放弃所有续传令牌
void sys_abandon_continuation(thread_t *thread) {
    if (thread->current_continuation) {
        continuation_release(thread->current_continuation);
        thread->current_continuation = NULL;
    }
}
```

### 5.4 Stop 不变量 (Stop Invariant)

```c
// 不变量：进程停止状态与线程状态的一致性
//     PROC_STOPPED  ⇒  所有线程 TASK_STOPPED
//     任何线程 TASK_RUNNING  ⇒  PROC_RUNNING

// 验证函数
bool validate_stop_invariant(process_t *proc) {
    if (proc->state == PROC_STOPPED) {
        // 所有线程必须处于 TASK_STOPPED 状态
        for_each_thread(thread, proc) {
            if (thread->state != TASK_STOPPED) {
                panic("Stop invariant violated: PROC_STOPPED but thread not TASK_STOPPED");
                return false;
            }
        }
    }
    return true;
}

// 设置进程停止状态
void set_process_stopped(process_t *proc) {
    // 先停止所有线程
    for_each_thread(thread, proc) {
        thread->state = TASK_STOPPED;
    }
    
    // 然后设置进程状态
    proc->state = PROC_STOPPED;
    
    // 验证不变量
    assert(validate_stop_invariant(proc));
}

// SIGSTOP 确认机制
void handle_stop_acknowledgment(process_t *proc, tid_t tid, uint64_t generation) {
    // 检查世代号是否匹配
    if (generation != proc->stop_generation) {
        // 过期的确认，忽略
        return;
    }
    
    // 减少等待计数
    atomic_dec(&proc->stop_pending_count);
    
    // 如果所有线程都已确认，设置进程停止状态
    if (atomic_read(&proc->stop_pending_count) == 0) {
        set_process_stopped(proc);
    }
}
```

### 5.5 Kill 不变量 (Kill Invariant)

```c
// 不变量：资源释放的安全性
//     free_page_tables() 之前：所有 CPU 已脱离 mm
//     free_tcb() 之前：线程已从运行队列移除

// 验证函数
bool validate_kill_invariant(process_t *proc) {
    // 检查所有 CPU 是否已脱离该进程的地址空间
    for_each_online_cpu(cpu) {
        thread_t *running = runqueues[cpu].curr;
        if (running && running->pid == proc->pid) {
            // 仍有 CPU 在运行该进程的线程
            panic("Kill invariant violated: CPU still in dying process mm");
            return false;
        }
    }
    return true;
}

// 安全释放流程
void safe_kill_process(process_t *proc) {
    // Phase 1: 标记所有线程为 TASK_DYING
    for_each_thread(thread, proc) {
        thread->state = TASK_DYING;
    }
    
    // Phase 2: 获取所有 runqueue lock
    for_each_online_cpu(cpu) {
        spin_lock(&runqueues[cpu].lock);
    }
    
    // Phase 3: 发送 IPI shootdown
    // 使用平台位图类型，支持超过 32 核
    cpumask_t cpu_mask;
    cpumask_clear(&cpu_mask);
    for_each_thread(thread, proc) {
        if (runqueues[thread->cpu].curr == thread) {
            cpumask_set_cpu(thread->cpu, &cpu_mask);
        }
    }
    
    // 初始化确认计数器
    atomic_set(&proc->ipi_ack_count, 0);
    
    for_each_cpu_in_mask(cpu, cpu_mask) {
        send_ipi(cpu, IPI_SHOOTDOWN);
    }
    
    // Phase 4: 释放 runqueue lock
    for_each_online_cpu(cpu) {
        spin_unlock(&runqueues[cpu].lock);
    }
    
    // Phase 5: 等待所有 CPU 确认
    while (atomic_read(&proc->ipi_ack_count) < cpumask_weight(&cpu_mask)) {
        cpu_pause();
    }
    
    // Phase 6: 验证不变量
    assert(validate_kill_invariant(proc));
    
    // Phase 7: 安全释放资源
    for_each_thread(thread, proc) {
        free_tcb(thread);
    }
    free_page_tables(proc);
}
```

### 5.6 COW 不变量 (Copy-on-Write Invariant)

```c
// 不变量：内存读取不改变地址空间状态
//     sys_process_read_memory() 执行前后：
//         目标进程的页表状态不变
//         目标进程的 COW 页不被触发
//         目标进程的 dirty 位不变

// 验证函数
bool validate_cow_invariant(process_t *proc, uint64_t addr, size_t len) {
    // 读取前记录页表状态
    struct page_state before[MAX_PAGES];
    record_page_states(proc, addr, len, before);
    
    // 执行读取
    void *buf = kmalloc(len);
    sys_process_read_memory(proc->pid, buf, (void *)addr, len);
    
    // 读取后检查页表状态
    struct page_state after[MAX_PAGES];
    record_page_states(proc, addr, len, after);
    
    // 比较状态
    for (int i = 0; i < nr_pages; i++) {
        if (before[i].pte != after[i].pte) {
            panic("COW invariant violated: page table changed after read");
            return false;
        }
        if (before[i].dirty != after[i].dirty) {
            panic("COW invariant violated: dirty bit changed after read");
            return false;
        }
    }
    
    kfree(buf);
    return true;
}
```

### 5.7 RIP 验证不变量 (RIP Validation Invariant)

```c
// 不变量：线程执行流的完整性
//     sys_thread_set_state_safe() 设置的 RIP 必须满足：
//         RIP 在用户空间范围内
//         RIP 指向已注册的 handler 或 trampoline 或 sigframe
//         RIP 符合 Canonical 规范

// 验证函数
bool validate_rip_invariant(thread_t *thread, uint64_t rip, enum RipValidationPolicy policy) {
    // 检查 RIP 是否在用户空间
    if (!is_user_address(rip)) {
        panic("RIP invariant violated: RIP not in user space");
        return false;
    }
    
    // 检查 Canonical 规范
    if (!is_canonical(rip)) {
        panic("RIP invariant violated: RIP not canonical");
        return false;
    }
    
    // 根据策略检查
    switch (policy) {
        case RIP_POLICY_HANDLER:
            // RIP 必须指向已注册的 handler
            if (!is_registered_handler(thread, rip)) {
                panic("RIP invariant violated: RIP not a registered handler");
                return false;
            }
            break;
            
        case RIP_POLICY_TRAMPOLINE:
            // RIP 必须指向 trampoline page
            if (!is_trampoline_address(rip)) {
                panic("RIP invariant violated: RIP not in trampoline page");
                return false;
            }
            break;
            
        case RIP_POLICY_SIGFRAME:
            // RIP 必须指向已验证的 sigframe 区域
            if (!is_valid_sigframe_address(thread, rip)) {
                panic("RIP invariant violated: RIP not in valid sigframe");
                return false;
            }
            break;
            
        case RIP_POLICY_EXCEPTION:
            // RIP 必须指向异常恢复地址
            if (!is_exception_recovery_address(rip)) {
                panic("RIP invariant violated: RIP not an exception recovery address");
                return false;
            }
            break;
    }
    
    return true;
}
```

### 5.8 不变量总结表

| 不变量名称 | 条件 | 保证 |
|-----------|------|------|
| AST 不变量 | `ast_depth > 0` | `sigexec_state != SIGEXEC_NONE` |
| Signal Delivery 不变量 | 信号投递中 | `ast_depth <= MAX_AST_DEPTH`，异常帧最多 1 个 |
| Restart 不变量 | `current_continuation != NULL` | 线程尚未执行新 syscall |
| Stop 不变量 | `PROC_STOPPED` | 所有线程 `TASK_STOPPED` |
| Kill 不变量 | `free_page_tables()` 前 | 所有 CPU 已脱离 mm |
| COW 不变量 | `sys_process_read_memory()` 后 | 页表状态不变 |
| RIP 不变量 | `sys_thread_set_state_safe()` 后 | RIP 在白名单中 |

---

## 6. 生命周期同步

### 6.1 Fork 二阶段提交与冻结窗口期

**Phase 1 (Freeze & Sync)**：
1. Process Server 拦截 `fork` 系统调用，向 Signal Server 发起同步阻塞的 `MSG_FORK_BEGIN_SYNC` 请求
2. Signal Server 收到请求后，将父进程的 `signal_freeze` 标志置为 `true`
3. 冻结窗口期到达的信号被定向存入父进程专属的 `freeze_pending_queue`
4. Signal Server 完成设置后返回确认
5. Process Server 呼叫微内核执行物理内存与 TCB 的克隆

**Phase 2 (Isolate & Resume)**：
1. 微内核克隆完成后返回 Process Server
2. Process Server 分配新的 PID，向 Signal Server 发送 `MSG_FORK_COMPLETE_SYNC`
3. Signal Server 对子进程的信号状态进行彻底清洗：
   - 显式清空子进程的 `standard_pending` 掩码
   - 清空子进程的所有 `rt_queues` 队列
   - 切断父进程未决信号的泄漏（代际隔离语义）
4. Signal Server 解除父子进程的 `signal_freeze` 状态
5. Signal Server 调用 `drain_and_deliver()` 将暂存队列中的信号严格路由至父进程
6. Process Server 负责将子进程的所有 continuation handle 强制清零（续传属于父进程的阻塞调用，子进程不继承）

**代际隔离语义**：子进程的信号状态在 fork 后必须与父进程完全隔离。父进程的未决信号、RT 信号队列、续传令牌均不继承到子进程，防止信号泄漏和 UAF。

### 6.2 进程终止协议 (waitpid 与 SIGCHLD 竞态)

POSIX Server 内部充当了 Linux 中 `Task struct` 的部分角色，维护**僵尸进程表**：

1. **子进程退出**：微内核发消息给 POSIX Server 告知退出码
2. **状态转移**：POSIX Server 将该进程转入 Zombie 状态，投递 `SIGCHLD`
3. **waitpid 消费**：父进程调用 `waitpid` 实则是向 POSIX Server 发送 IPC。若表中有 Zombie，立即出队返回；若无，父进程阻塞等待 IPC 回复

### 6.3 进程组与会话生命周期

POSIX Server 维护 pgrp 表和 session 表，支持 `SignalTarget` 中的 `TARGET_PGRP` 和 `TARGET_SESSION` 投递：

**新增 IPC 消息**：

| 消息 | 触发方 | 作用 |
|------|--------|------|
| `MSG_SETPGRP(pid, pgid)` | libc (`setpgid`) | 更新进程组映射 |
| `MSG_SETSID(pid)` | libc (`setsid`) | 创建新会话，更新会话映射 |
| `MSG_EXIT_PGRP(pgid)` | 微内核 | 进程组最后一个进程退出，清理映射 |
| `MSG_EXIT_SESSION(session_id)` | 微内核 | 会话最后一个进程退出，清理映射 |

**生命周期绑定**：
- `fork`：子进程继承父进程的 pgrp 和 session
- `exec`：pgrp 和 session 不变
- `exit`：进程从 pgrp 和 session 表中移除，若为空则清理
- `setpgid`/`setsid`：更新映射表，旧映射在无引用时回收

---

## 7. 架构优缺点总结

### 7.1 核心优势 (The Pros)

1. **顶级的调试生态 (Debugger-Friendly)**：异常即 IPC。调试器挂载异常端口，信号生成前即可完美拦截现场，告别 `ptrace`。

2. **极致的内核安全**：微内核代码消灭了位图与蹦床。即便 POSIX Server 发生数组越界，系统内核绝不 Panic。

3. **根治异步安全死锁 (Async-Signal-Safety Solved)**：信号仅在 AST 边界或明确系统调用处介入。告别"指令中途劈开"的噩梦，根除锁重入困扰。

4. **准确的 I/O 恢复**：`SyscallRestartInfo` 配合驱动 Cookie，彻底解决非幂等调用被打断的数据重复问题。

5. **最小权限原则与 RIP 验证**：Capability 细粒度拆分和 RIP 内存属性验证（基于页表，不依赖 POSIX 语义），即使 POSIX Server 被 exploit，也无法注入任意 RIP 执行任意代码，防止全系统 Ring 3 进程沦陷。

6. **COW 安全的内存读取**：`sys_process_read_memory` 保证不破坏 Copy-on-Write 语义，Core Dump 和调试器读取目标进程内存时不会改变程序行为，确保调试和取证的准确性。

### 7.2 工程挑战与缺点 (The Cons)

1. **高昂的 IPC 性能开销**：一次 `#PF` 涉及四次内核/用户态切换（`肇事线程 -> 微核 -> POSIX Server -> 微核 -> Handler`）。极度依赖微内核 Fastpath 寄存器传参优化。

2. **组件同步复杂度极高**：Token 失效、Fork 冻结窗口、端口超时降级必须严丝合缝，任意竞态都会导致 Use-After-Free 级别的逻辑漏洞。

3. **驱动心智负担**：所有的阻塞驱动都必须维护一套机制，识别 `restart_cookie` 并提供正确的 `bytes_completed`，大幅提高了驱动开发的门槛。

4. **get_user_pages 实现复杂度**：`sys_process_read_memory` 需要实现类似 Linux `get_user_pages` 的复杂机制，正确处理 COW、缺页、页表遍历等场景。这增加了微内核的代码复杂度，但这是保证 Core Dump 不破坏程序行为的必要代价。

---

## 8. 信号执行模型（Signal Execution Model）

本章汇总整个信号系统的执行语义，将分散在各处的隐含假设集中到一处，从"协议"角度重新描述。

### 8.1 投递边界语义（→ 见 2.8 节）

系统只在确定性边界投递信号，不在任意指令处注入。这是与 libc/runtime/JIT 的 ABI 合约。

### 8.2 协作式语义（Defer 机制与预算规则）

- `signal_defer_depth` 是用户态声明的临界区计数器
- SIGKILL 永不 defer（内核直达）
- SIGSTOP/调试器 defer 有超时上限
- SIGINT/SIGTERM/SIGUSR 可无限 defer

### 8.3 嵌套语义（MAX_AST_DEPTH 及递归检测）

- AST 栈最大深度为 8，投递前检查
- 异常递归（AST 栈中 `AST_EXCEPTION` 帧超过 1 个）视为不可恢复，强杀进程
- 嵌套信号在 AST 栈中逐帧保存，每帧是独立的 `SignalTransition` 对象

### 8.4 转换语义（SignalTransition 对象生命周期）

```
创建：ast_push() → 保存 interrupted 现场、info、reason、cookie
使用：POSIX Server 读取 SignalTransition 构建 sigframe
恢复：sigreturn → ast_pop() → 恢复 interrupted 现场 → 释放对象
```

- 每个 `SignalTransition` 包含完整的 `HardwareInterruptFrame`、`siginfo_t`、`ucontext_t`
- `cookie` 用于 SROP 防御，在 `ast_push` 时生成，`sigreturn` 时校验
- `reason` 字段区分信号来源（TRANS_SIGNAL/TRANS_EXCEPTION/TRANS_STOP/TRANS_DEBUG）

### 8.5 调度器交互（AST 检查点与 IPI 触发）

- AST 检查点位于 syscall 返回用户态、中断处理返回用户态、异常处理返回用户态
- `ast_pending` 是核心劫持标志，调度器在返回用户态前检查
- IPI 强制抢占用于卡在用户态死循环的线程，强制进入 kernel boundary 后走 AST 路径

### 8.6 运行时交互（libc/GC/JIT 如何使用 signal_defer_depth）

- **libc**：malloc/free 等敏感操作通过 `signal_defer_depth` 声明临界区
- **GC**：stop-the-world 期间可安全 defer，不受普通信号打断
- **JIT**：无需在生成的代码中插入信号检查点，依赖投递边界语义

### 8.7 调试器交互（Exception Port 订阅与信号优先级）

- 调试器挂载异常端口，在信号生成前即可拦截现场
- 调试器停止信号（`TRANS_DEBUG`）有 defer 超时上限，防止被无限延迟
- 异常端口优先级高于普通信号投递，确保调试器可控
