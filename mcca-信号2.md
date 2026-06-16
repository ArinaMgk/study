## Mecocoa

## 第一部分：三层解耦架构 (The 3-Tier Architecture)

整个信号系统被严格切分为三个物理隔离的层级，彻底避免宏内核的面条代码。

**1. 底层：Mecocoa 微内核 (Ring 0 - The Core)**

- **定位**：只提供“机制”，完全不懂 POSIX 标准，不知道什么是 SIGINT 或 SIGSEGV。
    
- **职责**：
    
    - 提供纯粹的 IPC 消息传递基础设施（包括异常端口）。
        
    - 提供 AST（异步系统陷阱）的调度器劫持点（仅修改现场，零 IPC）。
        
    - 提供基于 Capability Token 鉴权的线程现场篡改接口（读写内存/修改寄存器）。
        
    - 保留最高权限的 `sys_task_force_kill`。
        

**2. 中间层：POSIX 兼容服务 (Ring 3 - The POSIX Server)**

- **定位**：一个拥有特权的 Ring 3 用户进程，提供“策略”，是系统的信号大脑。
    
- **职责**：
    
    - 监听各级异常端口，将微内核发来的硬件异常错误码翻译为 POSIX 信号。
    
    - 维护所有进程和线程的 POSIX 状态机（`sigaction` 表、屏蔽掩码、RT 信号队列）。
    
    - 负责计算红区（Red Zone）、检查备用栈，构建 `SigStackFrame` 并通过 `sys_thread_set_state` 设置线程现场。
        

**3. 顶层：用户态应用库 (Ring 3 - libc / The Trampoline)**

- **定位**：普通用户态进程。
    
- **职责**：
    
    - 提供只读映射的 `AST_TRAMPOLINE_VADDR` 页，包含纯 `syscall` 指令（不触碰栈），供线程"自首"汇报状态。
        
    - 提供标准的 `__sigreturn` 存根，负责在 Handler 执行完毕后呼叫 POSIX Server 恢复现场。
        

---

## 第二部分：核心数据结构与鉴权模型

**1. 状态分离模型 (运行于 POSIX Server)**


```
struct RTSigEntry {
    siginfo_t info;
    uint64_t  enqueue_order;  // 入队序号（用于 FIFO 排序）
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
    
    ThreadSignalState threads[];          
};
```

**信号投递顺序策略（POSIX 兼容）**：

```
// 信号投递选择函数
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

**2. 核心控制块与安全鉴权令牌 (微内核侧)** 微内核调度器依赖 `ast_pending`，而跨进程修改依赖 `CapabilityToken`。


```
#define MAX_AST_DEPTH 8  // 嵌套信号最大深度

// AST 上下文类型：区分不同的可重入来源
enum AstContext {
    AST_NONE,       // 不在 AST 中
    AST_SIGNAL,     // 普通信号处理
    AST_SIGRETURN,  // sigreturn 执行中
    AST_EXCEPTION,  // 异常处理（#PF, #GP 等）
    AST_TRAMPOLINE, // 蹦床执行中
};

struct ThreadControlBlock {
    atomic_t    ast_pending;    // 核心劫持点
    bool        in_ast_checkpoint; // 线程是否处于 AST 检查点
    bool        stop_pending;   // SIGSTOP 合作式挂起标记
    bool        signal_deferred; // 信号延迟标记（critical section 内）
    bool        need_resched;   // 需要调度标志（IPI handler 设置）
    uint64_t    stop_generation; // 停止世代号（防止过期确认）
    int         ast_depth;      // 当前 AST 栈深度
    atomic_int  signal_defer_depth; // 信号延迟深度（用户态协作安全点）
    uint64_t    current_restart_cookie; // 当前线程持有的重启令牌
    uint32_t    flags;          // 线程标志（THREAD_FLAG_DYING 等）
    
    // AST 栈：支持嵌套信号处理，记录每次劫持的上下文类型
    struct {
        HardwareInterruptFrame pre_frame;  // 劫持前的完整硬件帧
        uint64_t sigreturn_cookie;         // SROP 防御：sigreturn 校验值
        enum AstContext context;           // 上下文类型：区分可重入来源
        int signo;                         // 触发的信号编号（用于检测 SIGSEGV 递归）
    } ast_stack[MAX_AST_DEPTH];
};

// 线程标志定义
#define THREAD_FLAG_DYING      (1 << 0)  // 线程正在死亡（SIGKILL shootdown）

struct CapabilityToken {
    uint64_t  opaque_id;      
    pid_t     target_pid;     
    uint32_t  permissions;    // 细粒度权限位图
    uint64_t  expiry_version; // 杜绝 Use-After-Free
};

// --- Capability 细粒度权限定义（最小权限原则） ---
// 原始 CAP_PERM_WRITE_STATE 权限过大，拆分为细粒度权限
#define CAP_PERM_READ_STATE        (1 << 0)  // 允许 sys_thread_get_state
#define CAP_PERM_SIGNAL_DELIVERY   (1 << 1)  // 允许信号投递（RIP 必须指向 registered handler）
#define CAP_PERM_EXCEPTION_RECOVER (1 << 2)  // 允许异常恢复（RIP 必须指向 registered handler）
#define CAP_PERM_SIGRETURN_RESTORE (1 << 3)  // 允许 sigreturn 恢复现场（RIP 必须通过验证）
#define CAP_PERM_READ_MEM          (1 << 4)  // 允许 sys_process_read_memory（必须不破坏 COW）
#define CAP_PERM_WRITE_MEM         (1 << 5)  // 允许 sys_process_write_memory（仅限 sigframe 区域）
#define CAP_PERM_FORCE_KILL        (1 << 6)  // 允许 sys_task_force_kill
#define CAP_PERM_SUSPEND           (1 << 7)  // 允许 sys_thread_suspend/resume

// --- RIP 验证策略（防止任意代码执行） ---
// POSIX Server 的 sys_thread_set_state 必须通过 RIP 验证
enum RipValidationPolicy {
    RIP_POLICY_HANDLER,      // RIP 必须指向已注册的信号 handler
    RIP_POLICY_TRAMPOLINE,   // RIP 必须指向内核提供的 trampoline page
    RIP_POLICY_SIGFRAME,     // RIP 必须指向已验证的 sigframe 区域
    RIP_POLICY_EXCEPTION,    // RIP 必须指向异常恢复地址
};

// --- sys_thread_set_state 的安全封装 ---
// 微内核提供的安全接口，强制执行 RIP 验证
long sys_thread_set_state_safe(
    tid_t tid,
    struct HardwareInterruptFrame *new_frame,
    enum RipValidationPolicy policy,
    uint64_t policy_data  // 根据 policy 不同，含义不同：
                          // - RIP_POLICY_HANDLER: signo
                          // - RIP_POLICY_TRAMPOLINE: 不使用
                          // - RIP_POLICY_SIGFRAME: sigframe 地址
                          // - RIP_POLICY_EXCEPTION: exception vector
);
```

**3. sys_thread_get_state 返回结构**

POSIX Server 通过此结构获取线程状态。当线程处于 AST 检查点时，附带栈顶的预劫持快照：

```
struct ThreadState {
    HardwareInterruptFrame  current;              // 当前执行状态（可能在 Trampoline 内部）
    
    // 仅当线程处于 AST 检查点时有效
    bool                    in_ast_checkpoint;
    int                     ast_depth;            // 当前 AST 栈深度
    HardwareInterruptFrame  ast_top_frame;        // 栈顶的劫持前现场
    uint64_t                ast_top_cookie;       // 栈顶的 sigreturn 校验值
};
```

POSIX Server 调用 `sys_thread_get_state(tid)` 后，根据 `state.in_ast_checkpoint` 判断：
- 若为 `true`：使用 `state.ast_top_frame` 填写 `ucontext_t`，这才是 `sigreturn` 后要恢复的完整现场
- 若为 `false`：使用 `state.current` 获取当前执行状态
---

## 第二部分补充：系统不变量 (System Invariants)

系统不变量是设计中必须始终保持成立的条件，对于正确性和安全性至关重要。任何代码路径都不得违反这些不变量。

### 1. AST 不变量 (AST Invariant)

```
// 不变量：AST 栈深度与检查点标志的一致性
// 任何时刻：
//     ast_depth > 0  ⇔  in_ast_checkpoint == true
//     ast_depth == 0 ⇔  in_ast_checkpoint == false

// 验证函数
bool validate_ast_invariant(thread_t *thread) {
    if (thread->ast_depth > 0 && !thread->in_ast_checkpoint) {
        // 违反不变量：有 AST 栈深度但未设置检查点标志
        panic("AST invariant violated: ast_depth > 0 but in_ast_checkpoint == false");
        return false;
    }
    if (thread->ast_depth == 0 && thread->in_ast_checkpoint) {
        // 违反不变量：无 AST 栈深度但设置了检查点标志
        panic("AST invariant violated: ast_depth == 0 but in_ast_checkpoint == true");
        return false;
    }
    return true;
}

// 压栈时必须同时设置
void ast_push(thread_t *thread, HardwareInterruptFrame *frame, int signo) {
    assert(thread->ast_depth < MAX_AST_DEPTH);
    thread->ast_stack[thread->ast_depth].pre_frame = *frame;
    thread->ast_stack[thread->ast_depth].signo = signo;
    thread->ast_depth++;
    thread->in_ast_checkpoint = true;  // 必须同时设置
}

// 出栈时必须同时清除
void ast_pop(thread_t *thread) {
    assert(thread->ast_depth > 0);
    thread->ast_depth--;
    if (thread->ast_depth == 0) {
        thread->in_ast_checkpoint = false;  // 必须同时清除
    }
}
```

### 2. Signal Delivery 不变量 (Signal Delivery Invariant)

```
// 不变量：一个线程同一时刻最多只有一个 active signal frame
// 这意味着：
//     同一时刻，最多只有一个 SigStackFrame 正在被处理
//     新信号投递前，必须等待前一个信号处理完成或被嵌套

// 验证函数
bool validate_signal_delivery_invariant(thread_t *thread) {
    // 检查 AST 栈中的信号数量
    int signal_count = 0;
    for (int i = 0; i < thread->ast_depth; i++) {
        if (thread->ast_stack[i].context == AST_SIGNAL) {
            signal_count++;
        }
    }
    
    // 信号数量应该等于 AST 深度（假设只有信号嵌套）
    // 或者更宽松的检查：信号数量 <= AST 深度
    if (signal_count > thread->ast_depth) {
        panic("Signal delivery invariant violated: more signals than AST depth");
        return false;
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

### 3. Restart 不变量 (Restart Invariant)

```
// 不变量：重启令牌的生命周期
//     current_restart_cookie != 0  ⇒  线程尚未执行新的 syscall
//     线程执行新的 syscall（非 sys_restart_syscall）⇒  current_restart_cookie = 0

// 验证函数
bool validate_restart_invariant(thread_t *thread) {
    // 如果有有效的 cookie，线程应该处于可重启状态
    if (thread->current_restart_cookie != 0) {
        // 检查线程是否在等待重启
        if (thread->state != TASK_INTERRUPTIBLE && 
            thread->state != TASK_RUNNING) {
            panic("Restart invariant violated: cookie exists but thread not in restartable state");
            return false;
        }
    }
    return true;
}

// 系统调用入口：清除旧 cookie
long syscall_entry_handler(thread_t *thread, long syscall_nr) {
    // 任何新的系统调用（非 sys_restart_syscall）都清除旧 cookie
    if (syscall_nr != SYS_restart_syscall) {
        thread->current_restart_cookie = 0;
    }
    
    // 继续处理系统调用...
}

// 设置重启 cookie
void set_restart_cookie(thread_t *thread, uint64_t cookie) {
    thread->current_restart_cookie = cookie;
}
```

### 4. Stop 不变量 (Stop Invariant)

```
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

### 5. Kill 不变量 (Kill Invariant)

```
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
    atomic_int cpu_mask = 0;
    for_each_thread(thread, proc) {
        if (runqueues[thread->cpu].curr == thread) {
            atomic_or(&cpu_mask, (1 << thread->cpu));
        }
    }
    
    for_each_cpu_in_mask(cpu, cpu_mask) {
        send_ipi(cpu, IPI_SHOOTDOWN);
    }
    
    // Phase 4: 释放 runqueue lock
    for_each_online_cpu(cpu) {
        spin_unlock(&runqueues[cpu].lock);
    }
    
    // Phase 5: 等待所有 CPU 确认
    while (atomic_read(&ack_count) < popcount(cpu_mask)) {
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

### 6. COW 不变量 (Copy-on-Write Invariant)

```
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

### 7. RIP 验证不变量 (RIP Validation Invariant)

```
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
            if (!is_registered_handler(thread->pid, rip)) {
                panic("RIP invariant violated: RIP not in handler whitelist");
                return false;
            }
            break;
        case RIP_POLICY_TRAMPOLINE:
            if (rip != AST_TRAMPOLINE_VADDR && rip != SIGRETURN_TRAMPOLINE_VADDR) {
                panic("RIP invariant violated: RIP not in trampoline page");
                return false;
            }
            break;
        case RIP_POLICY_SIGFRAME:
            if (!is_valid_sigframe(thread, rip)) {
                panic("RIP invariant violated: RIP not in valid sigframe");
                return false;
            }
            break;
    }
    
    return true;
}
```

**不变量总结表**：

| 不变量名称 | 条件 | 保证 |
|-----------|------|------|
| AST 不变量 | `ast_depth > 0` | `in_ast_checkpoint == true` |
| Signal Delivery 不变量 | 信号投递中 | 最多一个 active signal frame |
| Restart 不变量 | `current_restart_cookie != 0` | 线程尚未执行新 syscall |
| Stop 不变量 | `PROC_STOPPED` | 所有线程 `TASK_STOPPED` |
| Kill 不变量 | `free_page_tables()` 前 | 所有 CPU 已脱离 mm |
| COW 不变量 | `sys_process_read_memory()` 后 | 页表状态不变 |
| RIP 不变量 | `sys_thread_set_state_safe()` 后 | RIP 在白名单中 |

---

## 第三部分：三大核心工作流 (Core Workflows)

### 场景 A：硬件异常的拦截与翻译 —— 异常端口降级链

微内核屏蔽硬件细节，将其统一为 IPC 异常消息，并提供带超时的三级降级投递。

- **触发与冻结**：线程触发 `#PF` 等异常，微内核捕获并 `Suspended` 该线程。
    
- **带超时的三级降级**：
    
    - `sys_thread_bind_exception_port` 绑定时已配置超时。
        
    - 首选 `Thread Exception Port`（GDB）。超时无响应，降级至 `Task Exception Port`。再次超时，发往 `Host Default Port`（POSIX Server）。
        
- **翻译与篡改**：POSIX Server 翻译出 `SIGSEGV`，向微内核出示 Token 篡改现场。
    
- **放行**：微内核解冻线程，跳入 Handler。

**异常嵌套与 AST 上下文记录**：

当异常发生时，微内核需要记录当前的 AST 上下文类型，以便检测不可恢复状态：

```
// 异常处理入口
void exception_handler(int vector, HardwareInterruptFrame* frame) {
    thread_t *current = get_current_thread();
    
    // 检测异常嵌套：异常发生在 AST 处理中
    if (current->ast_depth > 0) {
        enum AstContext top_context = current->ast_stack[current->ast_depth - 1].context;
        
        // 异常发生在 sigreturn 中：严重错误，强杀
        if (top_context == AST_SIGRETURN) {
            sys_task_force_kill(current->pid, SIGKILL);
            return;
        }
        
        // 异常发生在 trampoline 中：严重错误，强杀
        if (top_context == AST_TRAMPOLINE) {
            sys_task_force_kill(current->pid, SIGKILL);
            return;
        }
        
        // Page fault 在信号 handler 中：可能是 handler 自身的 bug
        if (vector == PF_VECTOR && top_context == AST_SIGNAL) {
            // 记录为异常上下文，允许处理
            // 但如果再次嵌套则强杀
        }
    }
    
    // 记录异常上下文到 AST 栈（如果需要）
    if (current->ast_depth < MAX_AST_DEPTH) {
        current->ast_stack[current->ast_depth].context = AST_EXCEPTION;
        current->ast_stack[current->ast_depth].signo = translate_exception_to_signal(vector);
        // 注意：异常不压入完整帧，因为 POSIX Server 会处理
    }
    
    // 发送异常 IPC
    ipc_send_exception(current->exception_port, vector, frame);
}
```

**【支撑接口代码】**

```
// 进程/线程初始化时，绑定各级异常端口并严格配置超时
sys_thread_bind_exception_port(
    tid_t    tid,
    port_t   port,
    uint32_t timeout_ms,   // 超时阈值 (0=无限等待，通常给调试器或POSIX Server)
    port_t   fallback_port // 超时后的降级目标端口
);
```

### 场景 A+：SIG_DFL 默认动作表

当信号的目标 Handler 为 `SIG_DFL` 时，POSIX Server 按以下四类默认动作处理：

| 默认动作 | 代表信号 | POSIX Server 行为 |
|---------|---------|------------------|
| Terminate | SIGTERM, SIGINT, SIGHUP | 调用 `sys_task_force_kill`，设退出码 128+signo |
| Terminate + Core Dump | SIGQUIT, SIGSEGV, SIGILL, SIGABRT, SIGFPE | 先通知 Core Dump 服务（另一个 Ring 3 特权进程）写 coredump 文件，再调用 `sys_task_force_kill` |
| Ignore | SIGCHLD（默认）, SIGURG | 直接丢弃，不构建蹦床 |
| Stop | SIGSTOP, SIGTSTP, SIGTTIN, SIGTTOU | 采用合作式挂起，在 AST 边界安全挂起（见场景 F） |
| **内核原生** | **SIGKILL** | **微内核 Ring 0 直接拦截执行，不经过 POSIX Server（见场景 A++）** |

**Core Dump 服务**：独立的 Ring 3 特权进程，接收 POSIX Server 的 IPC 通知后，通过 `sys_process_read_memory` 读取目标进程地址空间，按 ELF coredump 格式写入文件系统。

**sys_process_read_memory 的 COW 安全约束**：

```
// 关键约束：read_memory MUST NOT alter target address space state
// 否则 dump 本身会改变程序行为，破坏调试和取证

// 错误的实现（会破坏 COW）：
long sys_process_read_memory_BAD(pid_t pid, void *buf, void *target_addr, size_t len) {
    process_t *proc = lookup_process(pid);
    
    // 错误：直接映射目标地址并读取
    // 这会触发 page fault，可能导致：
    // 1. fault in pages（分配新页）
    // 2. trigger COW（写时复制）
    // 3. dirty anonymous memory（弄脏匿名内存）
    void *kbuf = map_user_pages(proc, target_addr, len);
    copy_to_user(buf, kbuf, len);
    unmap_user_pages(kbuf);
    return 0;
}

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

// get_user_pages_safe 的实现要点
#define GUP_FLAGS_READ_ONLY    (1 << 0)  // 只读访问，不设置 dirty 位
#define GUP_FLAGS_NO_COW       (1 << 1)  // 不触发 COW，直接读取共享页
#define GUP_FLAGS_FAULT_IN     (1 << 2)  // 允许触发缺页（用于调试器）
#define GUP_FLAGS_NO_FAULT     (1 << 3)  // 不触发缺页（用于 coredump）

int get_user_pages_safe(process_t *proc, void *addr, size_t len,
                        struct page **pages, int flags) {
    // 遍历目标地址范围的页表
    for each page in range {
        pte_t *pte = get_pte(proc->mm, addr);
        
        if (!pte_present(*pte)) {
            // 页不在内存中
            if (flags & GUP_FLAGS_NO_FAULT) {
                // Coredump 场景：跳过未映射的页，填充 0
                pages[i] = ZERO_PAGE;
                continue;
            }
            
            if (flags & GUP_FLAGS_FAULT_IN) {
                // 调试器场景：允许触发缺页
                handle_mm_fault(proc->mm, addr);
            }
        }
        
        // 检查 COW 状态
        if (pte_write(*pte)) {
            // 页可写，直接获取
            pages[i] = pte_page(*pte);
            get_page(pages[i]);
        } else {
            // 页是 COW 的（只读副本）
            if (flags & GUP_FLAGS_NO_COW) {
                // 不触发 COW，直接读取共享页
                pages[i] = pte_page(*pte);
                get_page(pages[i]);
                // 注意：不设置 dirty 位，不修改页表
            } else {
                // 允许触发 COW（调试器场景）
                // 这会分配新页并复制内容
                pages[i] = do_cow_page(proc->mm, addr);
            }
        }
    }
    
    return nr_pages;
}
```

**Coredump 场景的特殊处理**：

```
// Coredump 写入时的页处理
void coredump_write_segment(int fd, process_t *proc, void *start, size_t len) {
    void *buf = kmalloc(PAGE_SIZE);
    
    for (uint64_t addr = (uint64_t)start; addr < (uint64_t)start + len; addr += PAGE_SIZE) {
        struct page *page;
        int ret = get_user_pages_safe(proc, (void *)addr, PAGE_SIZE, &page,
                                      GUP_FLAGS_READ_ONLY | GUP_FLAGS_NO_COW | GUP_FLAGS_NO_FAULT);
        
        if (ret < 0) {
            // 页未映射或无法访问，填充 0
            memset(buf, 0, PAGE_SIZE);
        } else {
            // 成功获取页，读取内容
            void *kaddr = kmap_atomic(page);
            memcpy(buf, kaddr, PAGE_SIZE);
            kunmap_atomic(kaddr);
            put_page(page);
        }
        
        // 写入 coredump 文件
        vfs_write(fd, buf, PAGE_SIZE);
    }
    
    kfree(buf);
}
```

**关键设计原则**：
- **只读访问**：`sys_process_read_memory` 必须是只读的，不设置 dirty 位
- **不触发 COW**：读取 COW 页时直接读取共享页，不分配新页
- **不触发缺页**：对于未映射的页，填充 0 而不是分配新页
- **不修改页表**：读取操作不应改变目标进程的任何页表状态
- **原子性**：使用 `kmap_atomic` 避免睡眠，确保读取过程不被中断

**Linux 参考**：
- `get_user_pages()` with `FOLL_FORCE | FOLL_DUMP`
- `FOLL_DUMP` 标志用于 coredump，跳过未映射的页
- `FOLL_FORCE` 标志用于调试器，允许读取不可读的页

### 场景 A++：SIGKILL 内核原生保底

**问题 1**：若 `SIGKILL` 的最终执行依赖 Ring 3 的 POSIX Server，一旦 POSIX Server 发生死锁、OOM 或被恶意进程利用 IPC 塞满消息队列，整个系统的 `SIGKILL` 就会失效，连管理员都无法杀掉恶意进程。系统的"活性（Liveness）"遭到破坏。

**问题 2（SMP CR3/TLB Tear-down Race）**：在 SMP 环境下，如果 CPU 0 调用 `free_page_tables` 和 `free_tcb` 释放进程资源，而 CPU 1 还在用户态执行该进程的线程，CPU 1 会因访问已释放的页表或 TCB 引发三重故障（Triple Fault），导致整个物理机重启。

**解决方案**：剥夺 POSIX Server 对 `SIGKILL` 的绝对控制权，改为微内核原生支持，并遵循严格的 SMP 屏障流程（Remote Stop & Quiescent Wait）。

**SIGKILL 处理流程**：

```
// kill(pid, SIGKILL) 系统调用入口
sys_kill_handler(pid_t target_pid, int sig) {
    if (sig == SIGKILL) {
        if (!check_capability(current, CAP_PERM_FORCE_KILL, target_pid))
            return -EPERM;
        
        process_t *proc = lookup_process(target_pid);
        
        // === Phase 1: 获取所有 runqueue lock 并打标记 ===
        // 必须先获取所有相关 CPU 的 runqueue lock，防止线程在检查期间 migrate
        cpu_set_t target_cpus;
        CPU_ZERO(&target_cpus);
        
        // 第一步：获取进程锁，标记所有线程为 DYING
        spin_lock(&proc->lock);
        proc->state = PROC_DYING;
        
        for_each_thread(t, proc) {
            t->state = TASK_DYING;
        }
        spin_unlock(&proc->lock);
        
        // 第二步：获取所有 CPU 的 runqueue lock（严格 SMP 同步）
        // 这防止任何线程在检查期间被调度或迁移
        for_each_online_cpu(cpu) {
            spin_lock(&runqueues[cpu].lock);
        }
        
        // 第三步：在持有所有 rq lock 的情况下，确定目标 CPU
        atomic_int cpu_mask = 0;
        atomic_int ack_count = 0;
        
        for_each_thread(t, proc) {
            // 在持有 rq lock 时检查，此时线程不可能被调度走
            if (t->state == TASK_DYING && t->cpu != current_cpu) {
                // 检查该 CPU 上是否真的在运行这个线程
                if (runqueues[t->cpu].curr == t) {
                    atomic_or(&cpu_mask, (1 << t->cpu));
                }
            }
        }
        
        // 第四步：在仍持有 rq lock 时发送 IPI
        // 这确保目标 CPU 在收到 IPI 前不会调度走该线程
        for_each_cpu_in_mask(cpu, cpu_mask) {
            send_ipi(cpu, IPI_SHOOTDOWN);
        }
        
        // 第五步：释放所有 runqueue lock（IPI 已发送，目标 CPU 会处理）
        for_each_online_cpu(cpu) {
            spin_unlock(&runqueues[cpu].lock);
        }
        
        // === Phase 2: 自旋等待 Quiescent ===
        // 等待所有 CPU 确认已离开该进程地址空间
        while (atomic_read(&ack_count) < popcount(cpu_mask)) {
            cpu_pause();  // 自旋等待
        }
        
        // === Phase 3: 安全释放 ===
        // 确认所有 CPU 都脱离后，才能释放资源
        for_each_thread(t, proc) {
            revoke_user_memory(t);
            free_tcb(t);
        }
        free_page_tables(proc);
        
        // 仅向 POSIX Server 发送异步死亡通知（Bookkeeping）
        ipc_notify_async(POSIX_SERVER, MSG_PROCESS_DIED, 
                         .pid = target_pid, 
                         .exit_code = 128 + SIGKILL);
        return 0;
    }
    
    // 其他信号走常规 POSIX Server 路径
    return ipc_forward(POSIX_SERVER, MSG_SIGNAL_SEND, ...);
}

// IPI Shootdown 处理函数（运行在被通知的 CPU 上）
// 注意：IPI handler 不能直接调用 schedule()，会导致 scheduler recursion、rq corruption
void ipi_shootdown_handler(void) {
    thread_t *current = get_current_thread();
    
    // 检查当前线程是否属于被强杀的进程
    if (current->state == TASK_DYING) {
        // 只设置 need_resched 标志，不直接调用 schedule()
        // 真正的调度发生在 interrupt return path
        current->need_resched = 1;
        current->flags |= THREAD_FLAG_DYING;  // 标记为正在死亡
    }
    
    // 发送确认（表示该 CPU 已收到 shootdown 请求）
    atomic_inc(&current->process->ack_count);
}

// Interrupt exit path（在 iret/sysret 之前执行）
// 这是真正进行调度的地方
void interrupt_exit_path(void) {
    thread_t *current = get_current_thread();
    
    // 检查是否需要调度
    if (current->need_resched || current->flags & THREAD_FLAG_DYING) {
        // 清除标志
        current->need_resched = 0;
        
        // 如果线程正在死亡，选择其他线程运行
        if (current->state == TASK_DYING) {
            // 从运行队列中移除
            dequeue_thread(current);
            // 选择下一个线程
            thread_t *next = pick_next_thread();
            context_switch(next);
        } else {
            // 正常调度
            schedule();
        }
    }
}
```

**关键设计原则**：
- **IPI handler 只设置标志**：不进行任何可能引起锁竞争或递归的操作
- **Interrupt exit path 调度**：在返回用户态或内核态之前检查标志并调度
- **避免 scheduler recursion**：不在 interrupt context 直接调用 schedule()

**POSIX Server 角色**：仅接收异步通知，更新内部状态（僵尸进程表），不参与实际执行。即使 POSIX Server 无响应，`SIGKILL` 仍能正常工作。

**注**：`SIGSTOP` 采用合作式挂起机制（见场景 F），在 AST 边界安全挂起，避免硬冻结导致的死锁。

### 场景 B：异步软信号投递 —— 自首式 AST 机制

- **发信与打标记**：POSIX Server 收到 `SIGINT` 请求，发 IPC 令目标线程 `ast_pending = 1`。

- **IPI 强制抢占**：若目标线程正在另一个 CPU 核心上运行（用户态死循环等场景），微内核立即向该 CPU 发送核间中断（IPI）。目标 CPU 收到 IPI 后被强制打断用户态执行，陷入 Ring 0。IPI 处理函数无需特殊逻辑，直接 `iret`/`sysret` 返回。由于必然经过"返回用户态边界"，AST 检查被立刻触发，实现受控异步抢占。

- **调度器劫持（零 IPC）**：微内核调度器准备返回用户态时：
    
    
    ```
    if (atomic_read(&thread->ast_pending)) {
        // 嵌套信号检查：超过最大深度直接强杀
        if (thread->ast_depth >= MAX_AST_DEPTH) {
            sys_task_force_kill(current->pid, SIGSEGV);
            return;
        }
        
        // 可重入来源检测：SIGSEGV 递归立即强杀
        int pending_signo = get_pending_signal(thread);
        if (pending_signo == SIGSEGV && thread->ast_depth > 0) {
            // 检查栈上是否已有 SIGSEGV
            for (int i = 0; i < thread->ast_depth; i++) {
                if (thread->ast_stack[i].signo == SIGSEGV) {
                    // SIGSEGV during SIGSEGV：不可恢复，立即强杀
                    sys_task_force_kill(current->pid, SIGKILL);
                    return;
                }
            }
        }
        
        // 压入 AST 栈，记录上下文类型
        thread->ast_stack[thread->ast_depth].pre_frame = *frame;
        thread->ast_stack[thread->ast_depth].sigreturn_cookie = generate_random_cookie();
        thread->ast_stack[thread->ast_depth].context = AST_SIGNAL;
        thread->ast_stack[thread->ast_depth].signo = pending_signo;
        thread->ast_depth++;
        thread->in_ast_checkpoint = true;
        frame->hw_eip = AST_TRAMPOLINE_VADDR;    // 指向只读蹦床
        frame->pusha_rdi = thread->tid;          // 参数传 tid
        atomic_set(&thread->ast_pending, 0);
    }
    ```
    
- **自首报告**：线程执行 `syscall` 陷入微内核（`SYS_AST_REPORT`），微内核将其挂起并通知 POSIX Server。全程不触碰用户态栈，保护 Red Zone。
    
- **构建与 ABI 传参**：POSIX Server 调用 `sys_thread_get_state(tid)` 获取 `ThreadState`，检查 `state.in_ast_checkpoint` 标志：
  - 若为 `true`，使用 `state.ast_top_frame` 作为完整现场来构建 `ucontext_t`
  - 严格按照 ABI 设置参数寄存器：
    

**【支撑逻辑代码】**

```
// x86-64 System V ABI 传参逻辑
frame->rdi = signo;
if (sa_flags & SA_SIGINFO) {
	frame->rsi = (uint64_t)&sig_frame->si; // siginfo_t*
	frame->rdx = (uint64_t)&sig_frame->uc; // ucontext_t*
}

// SROP 防御：从 AST 栈顶获取 cookie 并写入 SigStackFrame
sig_frame->cookie = state.ast_top_cookie;
```

**【RIP 验证与安全接口】**

POSIX Server 在调用 `sys_thread_set_state_safe` 时，必须指定验证策略，微内核会强制验证 RIP：

```
// POSIX Server 投递信号时的安全调用
long deliver_signal(tid_t tid, int signo, struct HardwareInterruptFrame *handler_frame) {
    // 获取目标进程的信号 handler 信息
    struct sigaction *sa = get_sigaction(tid, signo);
    
    // 验证 handler 地址
    if (sa->sa_handler == SIG_DFL || sa->sa_handler == SIG_IGN) {
        // 默认动作，不需要 RIP 验证
        return handle_default_action(tid, signo);
    }
    
    // 设置 handler_frame 的 RIP 为已注册的 handler
    handler_frame->rip = (uint64_t)sa->sa_handler;
    
    // 调用安全接口，指定 RIP 必须指向已注册的 handler
    return sys_thread_set_state_safe(
        tid,
        handler_frame,
        RIP_POLICY_HANDLER,  // RIP 验证策略
        signo                 // policy_data: 信号编号
    );
}

// 微内核的 RIP 验证实现
long sys_thread_set_state_safe(
    tid_t tid,
    struct HardwareInterruptFrame *new_frame,
    enum RipValidationPolicy policy,
    uint64_t policy_data
) {
    thread_t *thread = lookup_thread(tid);
    
    // 权限检查：必须有对应的 Capability
    if (!check_capability(current, get_required_cap(policy), thread->pid)) {
        return -EPERM;
    }
    
    // RIP 验证：根据策略执行不同的验证
    uint64_t target_rip = new_frame->rip;
    
    switch (policy) {
        case RIP_POLICY_HANDLER: {
            // 验证 RIP 指向已注册的信号 handler
            int signo = (int)policy_data;
            struct sigaction *sa = get_sigaction_from_kernel(thread, signo);
            
            // 微内核维护一个 handler 地址白名单
            if (!is_registered_handler(thread->pid, target_rip)) {
                // RIP 不在白名单中，拒绝执行
                log_security_violation(thread, "RIP not in handler whitelist");
                return -EINVAL;
            }
            
            // 额外检查：RIP 必须在用户空间范围内
            if (!is_user_address(target_rip)) {
                return -EINVAL;
            }
            break;
        }
        
        case RIP_POLICY_TRAMPOLINE: {
            // 验证 RIP 指向内核提供的 trampoline page
            if (target_rip != AST_TRAMPOLINE_VADDR && 
                target_rip != SIGRETURN_TRAMPOLINE_VADDR) {
                log_security_violation(thread, "RIP not in trampoline page");
                return -EINVAL;
            }
            break;
        }
        
        case RIP_POLICY_SIGFRAME: {
            // 验证 RIP 指向已验证的 sigframe 区域
            uint64_t sigframe_addr = policy_data;
            
            // 检查 sigframe 是否在用户栈上
            if (!is_valid_sigframe(thread, sigframe_addr)) {
                log_security_violation(thread, "Invalid sigframe address");
                return -EINVAL;
            }
            
            // RIP 应该指向 sigframe 中的返回地址位置
            if (target_rip != sigframe_addr && 
                target_rip != sigframe_addr + 8) {
                log_security_violation(thread, "RIP not in sigframe region");
                return -EINVAL;
            }
            break;
        }
        
        case RIP_POLICY_EXCEPTION: {
            // 验证 RIP 指向异常恢复地址
            int vector = (int)policy_data;
            
            // 微内核维护异常恢复地址
            if (!is_valid_exception_recovery_addr(thread, vector, target_rip)) {
                log_security_violation(thread, "Invalid exception recovery address");
                return -EINVAL;
            }
            break;
        }
        
        default:
            return -EINVAL;
    }
    
    // 所有验证通过，执行状态设置
    return do_thread_set_state(thread, new_frame);
}

// Capability 权限映射
uint32_t get_required_cap(enum RipValidationPolicy policy) {
    switch (policy) {
        case RIP_POLICY_HANDLER:
            return CAP_PERM_SIGNAL_DELIVERY | CAP_PERM_EXCEPTION_RECOVER;
        case RIP_POLICY_TRAMPOLINE:
            return CAP_PERM_SIGNAL_DELIVERY;
        case RIP_POLICY_SIGFRAME:
            return CAP_PERM_SIGRETURN_RESTORE;
        case RIP_POLICY_EXCEPTION:
            return CAP_PERM_EXCEPTION_RECOVER;
        default:
            return 0;  // 无权限
    }
}
```

**安全设计原则**：
- **最小权限原则**：拆分原始的 `CAP_PERM_WRITE_STATE` 为细粒度权限
- **RIP 白名单验证**：RIP 必须指向已注册的 handler 或内核提供的地址
- **防止任意代码执行**：即使 POSIX Server 被 exploit，也无法注入任意 RIP
- **审计日志**：所有验证失败都记录安全事件

**【强制进入 Kernel Boundary 的统一机制】**

**问题**：SYS_AST_REPORT 依赖 syscall 让线程"自首"，但如果线程卡在用户态死循环、cli-like userspace spin、AVX512 超长计算、TSX transaction、SMAP-heavy loop 等场景，AST 只能靠 IPI → interrupt return 才能进入 trampoline。某些架构下 userspace interrupt masking、lazy interrupt window、virtualization delay 会让 signal latency 巨大。

**解决方案**：采用多层强制机制，确保线程最终进入 kernel boundary。

```
// === Layer 1: IPI 强制抢占（立即响应） ===
void set_ast_pending(thread_t *thread) {
    atomic_set(&thread->ast_pending, 1);
    
    // 若线程正在另一个 CPU 上运行，发送 IPI 强制抢占
    if (thread->state == THREAD_RUNNING && thread->cpu != current_cpu) {
        send_ipi(thread->cpu, IPI_AST_KICK);
    }
}

// IPI 处理函数（极简，仅触发 AST 边界检查）
void ipi_ast_kick_handler(void) {
    // 无需任何操作，直接返回
    // 返回用户态时，调度器出口的 AST 检查会自动执行
    return;
}

// === Layer 2: Timer Tick 检查（兜底机制） ===
// 对于长时间不响应 IPI 的线程，timer tick 会定期检查
void timer_tick_handler(void) {
    thread_t *current = get_current_thread();
    
    // 检查是否有待处理的 AST
    if (atomic_read(&current->ast_pending)) {
        // 设置 need_resched，在 tick 返回时触发调度
        current->need_resched = 1;
    }
    
    // 更新运行时间，可能触发时间片耗尽
    update_runtime(current);
    if (current->runtime >= current->timeslice) {
        current->need_resched = 1;
    }
}

// === Layer 3: irq_work 机制（延迟工作队列） ===
// 对于需要在硬中断上下文之外执行的操作
struct irq_work {
    atomic_t pending;
    void (*func)(struct irq_work *work);
};

void irq_work_queue(struct irq_work *work) {
    if (atomic_cmpxchg(&work->pending, 0, 1) == 0) {
        // 新工作项，触发 IPI
        send_ipi(current_cpu, IPI_IRQ_WORK);
    }
}

void ipi_irq_work_handler(void) {
    // 处理 irq_work 队列
    process_irq_work_queue();
}

// === Layer 4: 特殊情况处理 ===

// TSX (Intel Transactional Synchronization Extensions) 处理
// TSX transaction 期间中断会被延迟
void handle_tsx_abort(thread_t *thread) {
    // 如果线程在 TSX transaction 中且有待处理信号
    if (thread->in_tsx && atomic_read(&thread->ast_pending)) {
        // 强制中止 transaction
        xabort(0xff);  // TSX abort code
    }
}

// 用户态中断屏蔽处理
// 某些架构允许用户态临时屏蔽中断（如 x86 的 STI/CLI 在 CPL=0）
void handle_userspace_interrupt_masking(thread_t *thread) {
    // 检查是否有长时间屏蔽中断的用户态代码
    if (thread->userspace_irq_masked_duration > MAX_IRQ_MASK_DURATION) {
        // 强制注入异常，打破屏蔽
        inject_exception(thread, EX_BREAKPOINT);
    }
}

// === 统一的 Kernel Boundary 入口检查 ===
// 所有返回用户态的路径都必须检查 AST
void return_to_userspace_check(thread_t *thread) {
    // 检查 AST
    if (atomic_read(&thread->ast_pending)) {
        // 触发 AST 路径
        trigger_ast_injection(thread);
        return;
    }
    
    // 检查调度
    if (thread->need_resched) {
        schedule();
        return;
    }
    
    // 检查信号延迟（用户态协作安全点）
    if (thread->signal_deferred && atomic_read(&thread->signal_defer_depth) == 0) {
        thread->signal_deferred = false;
        atomic_set(&thread->ast_pending, 1);
        trigger_ast_injection(thread);
        return;
    }
}
```

**多层机制总结**：

| 机制 | 触发条件 | 延迟 | 适用场景 |
|------|---------|------|---------|
| IPI 强制抢占 | `ast_pending` 设置时 | 微秒级 | 正常运行的线程 |
| Timer Tick | 定期（1-10ms） | 毫秒级 | 长时间不响应 IPI |
| irq_work | 需要延迟处理的工作 | 微秒级 | 硬中断上下文之外的操作 |
| TSX Abort | TSX transaction 中 | 立即 | Intel TSX 场景 |
| Breakpoint 注入 | 用户态中断屏蔽超时 | 立即 | 异常场景 |

**注**：Linux 使用 timer tick、resched IPI、irq_work、TIF_SIGPENDING 混合机制。本设计采用类似的多层策略，确保线程最终进入 kernel boundary。

```
// 1. 微内核调度器出口 (准备返回 Ring 3 前的最后一步，纯本地修改，零 IPC)
#define MAX_AST_DEPTH 8

enum AstContext {
    AST_NONE,
    AST_SIGNAL,
    AST_SIGRETURN,
    AST_EXCEPTION,
    AST_TRAMPOLINE,
};

struct ThreadControlBlock {
    atomic_t    ast_pending;
    bool        in_ast_checkpoint;
    bool        stop_pending;
    bool        signal_deferred;
    bool        need_resched;
    uint64_t    stop_generation;
    int         ast_depth;
    atomic_int  signal_defer_depth;
    uint64_t    current_restart_cookie;
    uint32_t    flags;
    struct {
        HardwareInterruptFrame pre_frame;
        uint64_t sigreturn_cookie;
        enum AstContext context;
        int signo;
    } ast_stack[MAX_AST_DEPTH];
};

// 调度器劫持时
if (atomic_read(&thread->ast_pending)) {
    if (thread->stop_pending) {
        // SIGSTOP 合作式挂起路径（见场景 F）
        ...
    } else if (atomic_read(&thread->signal_defer_depth) > 0) {
        // 用户态协作安全点：在 critical section 内，延迟注入
        thread->signal_deferred = true;
        atomic_set(&thread->ast_pending, 0);
    } else {
        // 嵌套信号检查：超过最大深度直接强杀
        if (thread->ast_depth >= MAX_AST_DEPTH) {
            sys_task_force_kill(current->pid, SIGSEGV);
            return;
        }
        
        // 可重入来源检测：SIGSEGV 递归立即强杀
        int pending_signo = get_pending_signal(thread);
        if (pending_signo == SIGSEGV && thread->ast_depth > 0) {
            for (int i = 0; i < thread->ast_depth; i++) {
                if (thread->ast_stack[i].signo == SIGSEGV) {
                    sys_task_force_kill(current->pid, SIGKILL);
                    return;
                }
            }
        }
        
        // 常规信号投递路径：压入 AST 栈
        thread->ast_stack[thread->ast_depth].pre_frame = *frame;
        thread->ast_stack[thread->ast_depth].sigreturn_cookie = generate_random_cookie();
        thread->ast_stack[thread->ast_depth].context = AST_SIGNAL;
        thread->ast_stack[thread->ast_depth].signo = pending_signo;
        thread->ast_depth++;
        thread->in_ast_checkpoint = true;
        frame->hw_eip = AST_TRAMPOLINE_VADDR;
        frame->rdi    = thread->tid;
    }
    atomic_set(&thread->ast_pending, 0);
}
```

Trampoline 在"自首" IPC 中携带 `tid`，POSIX Server 通过 `sys_thread_get_state(tid)` 获取 `ThreadState`，使用 `state.ast_top_frame` 作为 `ucontext` 中的恢复地址。

```
; 2. AST_TRAMPOLINE_VADDR (用户态汇编)
; 注意：蹦床内绝对不能操作栈（不许 PUSH/CALL），否则破坏 Red Zone
AST_TRAMPOLINE:
    mov rax, SYS_AST_REPORT    ; 特殊的系统调用号
    syscall                    ; 陷入内核，绝不碰 RSP
    ud2                        ; 防御性编程，永远不应执行到此
```

**ABI 安全说明**：

x86-64 System V ABI 规定栈顶以下 128 字节是"红区（Red Zone）"，编译器会把局部变量放在那里。若蹦床执行 `CALL`（会压栈），会碾碎红区里的局部变量。此外，栈不满足 16 字节对齐会导致 SSE/AVX 指令触发 `#GP` 崩溃。

**解决方案**：蹦床只包含纯粹的 `syscall` 陷阱指令，不触碰用户态栈。线程陷入微内核后被挂起，POSIX Server 通过 `sys_thread_set_state` 恢复现场，`ud2` 永远不会被执行。

**SYS_AST_REPORT 语义约束**：

微内核收到 `SYS_AST_REPORT` 系统调用后，将线程挂起并通知 POSIX Server。POSIX Server 在决定线程去向之前，**必须**先调用 `sys_thread_set_state` 设置目标帧，否则线程行为未定义。两种正确处理路径：

- **有信号投递**：`sys_thread_set_state` 设 `RIP = Handler`，内核恢复线程时使用修改后的帧 → 线程落入 Handler ✓
- **无信号可投递**（信号已被屏蔽等）：`sys_thread_set_state` 设 `RIP/RSP = ast_top_frame` 的值（还原原始现场），再唤醒线程 → 线程回到被劫持前的位置 ✓
- **错误做法**：直接唤醒线程而不调 `sys_thread_set_state` → 线程从蹦床末尾执行 `ud2`，触发 `#UD` 异常 ✗

### 场景 B+：用户态协作安全点 —— 真正的安全注入

**问题**：当前设计假设"返回用户态边界"就是"安全点"，但这并不成立。线程可能在 AST 边界持有用户态锁：
- 用户态 mutex
- runtime lock（JVM、Python、Rust async runtime）
- malloc arena / jemalloc / tcmalloc critical section
- glibc FILE lock
- language runtime GC critical section
- JIT patch code
- userspace RCU read-side critical section

此时在 AST 边界硬插 Handler，Handler 又调用 libc 尝试拿同一把锁，立刻死锁。这就是传统 Unix "async-signal-safe" 地狱。

**当前设计的局限性**：
- 实现的是"deferred asynchronous signal"（延迟异步信号）
- 不是"safe signal"（安全信号）
- 只是把问题概率降低，并非根除

**解决方案**：引入用户态协作安全点机制，让线程主动声明"现在允许异步 signal 注入"。

**1. 新增系统调用 `sys_signal_checkpoint`**：

```
// 用户态主动检查点
// 返回值：0 表示无待处理信号，非 0 表示信号已投递
long sys_signal_checkpoint(void) {
    thread_t *current = get_current_thread();
    
    // 检查是否有待处理的信号
    if (atomic_read(&current->ast_pending)) {
        // 触发 AST 路径，但此时线程已主动声明安全
        return trigger_ast_injection(current);
    }
    return 0;
}
```

**2. TCB 扩展：信号延迟计数器**：

```
struct ThreadControlBlock {
    // ... 其他字段
    atomic_int signal_defer_depth;  // 信号延迟深度
};
```

**3. 用户态 API**：

```
// 进入 critical section
void signal_defer_begin(void) {
    atomic_inc(&current_tcb->signal_defer_depth);
}

// 离开 critical section，检查点
void signal_defer_end(void) {
    if (atomic_dec_return(&current_tcb->signal_defer_depth) == 0) {
        sys_signal_checkpoint();  // 主动检查点
    }
}

// 便捷宏
#define SIGNAL_SAFE_BLOCK(code) \
    do { \
        signal_defer_begin(); \
        code; \
        signal_defer_end(); \
    } while(0)
```

**4. 调度器劫持逻辑更新**：

```
// 调度器出口 AST 检查
if (atomic_read(&thread->ast_pending)) {
    // 检查是否在 critical section 内
    if (atomic_read(&thread->signal_defer_depth) > 0) {
        // 在 critical section 内，不注入，设置延迟标记
        thread->signal_deferred = true;
        atomic_set(&thread->ast_pending, 0);
    } else {
        // 安全点，正常注入
        // ... 原有 AST 栈压入逻辑
    }
}
```

**5. 运行时库集成示例**：

```
// glibc malloc 示例
void* malloc(size_t size) {
    signal_defer_begin();  // 进入 critical section
    
    void* ptr = /* ... malloc 实现 ... */;
    
    signal_defer_end();    // 离开 critical section，检查点
    return ptr;
}

// pthread_mutex_lock 示例
int pthread_mutex_lock(pthread_mutex_t* mutex) {
    signal_defer_begin();
    int ret = /* ... lock 实现 ... */;
    signal_defer_end();
    return ret;
}

// JVM GC critical section
void gc_safepoint() {
    signal_defer_begin();
    // ... GC 操作 ...
    signal_defer_end();  // GC 结束后检查点，可能触发信号
}
```

**优势**：
- **真正解决 async-signal-safe 问题**：信号只在用户态主动声明的安全点注入
- **运行时库可控**：malloc、mutex、GC 等关键区域可以延迟信号注入
- **向后兼容**：不使用协作机制的程序行为与原来一致
- **渐进式采用**：可以逐步在关键运行时库中添加支持

**注**：此机制为可选增强，不强制要求。未使用协作机制的程序仍按原有 AST 边界注入，只是存在理论上的死锁风险（与传统 Unix 相同）。

### 场景 C：阻塞 IPC/系统调用的打断与重启 —— 状态化重启 (Stateful Thunks)

1.  **打断休眠**：信号唤醒阻塞在微内核的线程。
2.  **微内核暴露进度**：返回 `SyscallRestartInfo`，包含 `restart_cookie` (不透明令牌) 和 `bytes_completed` (已完成字节)。
3.  **时空接力**：若决定 `SA_RESTART`，libc 检测到重启标识后调用特供的 `sys_restart_syscall(cookie, bytes)`。微内核凭 cookie 路由给底层驱动，从断点处无缝续传。

**【支撑数据结构代码】**

对齐 Linux 的 `sys_restart_syscall` 模式，将重启语义**下沉为一个独立的系统调用**：

```
// 微内核在打断系统调用时向 POSIX Server 暴露的进度描述符
struct SyscallRestartInfo {
    enum RestartPolicy policy; // 如 RESTART_NONE, RESTART_VIA_THUNK
    uintptr_t syscall_ip;      // 准确的系统调用指令原始地址
    uint64_t  syscall_args[6]; // 原始参数快照
    
    // 进度与续传追踪（驱动层必须填写）
    uint64_t  bytes_completed; // 已消费/写入的字节数
	uint64_t restart_cookie; // 微内核向驱动服务颁发的不透明令牌 // 驱动服务凭此令牌知道从哪里续传
};
```

**重启的完整调用链**：

```
sigreturn 执行完毕
    ↓
libc 检测到 SigStackFrame 中有 RestartInfo（policy != RESTART_NONE）
    ↓
libc 调用 sys_restart_syscall(restart_cookie, bytes_completed)
    ↓
微内核持有 cookie → 找到当初被打断的 IPC → 路由给对应驱动服务
    ↓
驱动服务凭 cookie 找到自己的内部状态，从 bytes_completed 处继续
```

重启逻辑仅存在于微内核内核服务之间

**restart_cookie 的失效与清理**

`restart_cookie` 在以下场景失效，微内核必须主动清理：

- **exec()**：进程执行 `exec` 时，微内核作废该进程所有未完成的 `restart_cookie`：

```
// MSG_EXEC 协议流程
// 1. POSIX Server 重置 handlers
// 2. 微内核作废该进程所有 restart_cookie
msg_send(POSIX_SERVER, MSG_EXEC, target_pid);
kernel_invalidate_all_cookies(target_pid);  // 维护 per-process cookie 表，exec 时全部删除
```

- **fd 关闭**：Handler 内关闭了同一个 fd，驱动服务收到续传请求时自行返回 `-EBADF`，无需微内核介入。

- **siglongjmp 逃逸（混乱代理漏洞）**：线程在信号 Handler 里调用 `siglongjmp` 强行跳出信号上下文（不调用 `sigreturn`），Cookie 残留在内核。若线程关闭了原有 fd 又打开新文件复用该 fd，再执行重启调用，驱动会拿着旧 Cookie 向新目标写入数据，导致灾难性串键。

**解决方案（生命周期强绑定与即时作废）**：

确立一条铁律：**一个线程同时最多只能持有一个 Cookie，且任何新的系统调用都会作废它**。

1. 在微内核的 TCB 中记录 `current_restart_cookie`：

```
struct ThreadControlBlock {
    // ... 其他字段
    uint64_t current_restart_cookie;  // 当前线程持有的重启令牌
};
```

2. 当线程从用户态发起**任何新的系统调用**（而非 `sys_restart_syscall`）时，微内核自动将 TCB 中的 `current_restart_cookie` 清零：

```
// 系统调用入口（通用处理）
sys_call_entry_handler(syscall_num, args...) {
    // 非 restart_syscall 的任何系统调用都会作废旧 Cookie
    if (syscall_num != SYS_restart_syscall) {
        current->current_restart_cookie = 0;
    }
    // ... 正常系统调用处理
}
```

3. `siglongjmp` 必然会调用后续的其他系统调用，因此旧 Cookie 会被自动作废。驱动在查不到有效 Cookie 时，直接返回 `-EINTR`。

### 场景 D：同步信号主动消费 (sigsuspend / sigwaitinfo)

并非所有信号都是“被动打断”，同步 API 拥有独立路径，无需 AST 蹦床。
1.  **主动休眠**：线程发 IPC 给 POSIX Server：“我调用了 `sigwaitinfo`，等 `set` 中的信号”。
2.  **状态标记**：POSIX Server 将该线程标记为 `SIGNAL_WAIT`，暂停向其异步投递。
3.  **直接回复**：目标信号到达时，POSIX Server 直接将 `siginfo_t` 打包在 IPC Reply 中唤醒该线程。线程直接拿到结果，不修改任何 EIP。

### 场景 E：`sigreturn` 的现场恢复闭环

为保持微内核纯洁性，采用**推荐方案 B（POSIX Server 恢复）**。

**1. SigStackFrame 帧布局约定**

Handler 入口时的栈顶布局：

```
Handler 入口时的栈顶 (RSP):
  [8 bytes] __sigreturn 地址  ← RSP 指向这里 (作为返回地址)
  [N bytes] SigStackFrame     ← RSP + 8 开始
```

`__sigreturn` 存根执行 `ret` 指令后，RSP 弹出返回地址，此时 `RSP` 直接指向 `SigStackFrame` 的起始位置。因此：

```
SigStackFrame 地址 = sigreturn 陷入内核时的 RSP
```

**SigStackFrame 结构体定义**：

```
struct SigStackFrame {
    ucontext_t  uc;
    siginfo_t   si;
    alignas(64) uint8_t fpu_state[XSAVE_AREA_SIZE];  // FPU/SIMD 状态
    uint64_t    cookie;                              // SROP 防御：随机校验值
    
    // 注：__sigreturn 返回地址不在此结构体内
    // 它在 SigStackFrame 起始地址 - 8 处（即 new_rsp）
};
```

**2. 恢复流程**

1.  用户 Handler 结束，跳入 libc 的 `__sigreturn` 存根。
2.  存根执行系统调用陷入 Ring 0，微内核的 `sys_sigreturn_handler` 执行：

```
sys_sigreturn_handler:
    rsp_at_sigreturn = thread->user_frame.rsp   // 读取陷入时的 RSP
    ipc_notify(POSIX_SERVER, 
               MSG_SIGRETURN, 
               .tid = current_tid, 
               .user_rsp = rsp_at_sigreturn)    // 把 RSP 交给 POSIX Server
    thread_suspend(current)                     // 挂起线程，等待 POSIX Server 恢复现场
```

3.  POSIX Server 收到 `MSG_SIGRETURN` 通知，根据 `user_rsp` 定位 `SigStackFrame`（`SigStackFrame 地址 = user_rsp`），通过 `sys_process_read_memory` 读取帧内容。
4.  **SROP 防御**：POSIX Server 校验 `SigStackFrame.cookie` 是否与 AST 栈顶记录的 `sigreturn_cookie` 一致。不一致则调用 `sys_task_force_kill` 强杀进程（防止伪造栈帧攻击）。
5.  **现场有效性验证**：POSIX Server 在恢复现场前，执行严格的有效性验证：

```
// RIP 规范检查：必须在用户地址空间范围内，且符合 Canonical 规范
bool is_canonical(uint64_t addr) {
    // x86-64 Canonical 地址：高 16 位为全 0 或全 1（符号扩展）
    uint64_t high_bits = addr & 0xFFFF800000000000ULL;
    return (high_bits == 0 || high_bits == 0xFFFF800000000000ULL);
}

if (!is_canonical(frame->rip) || frame->rip >= USER_MAX_ADDR) {
    sys_task_force_kill(pid, token);  // RIP 非法，强杀
    return;
}

// XSAVE 清洗：强制将特性掩码与硬件实际支持的掩码做按位与
uint64_t hardware_xsave_mask = get_cpu_xsave_mask();
frame->xsave_header.xstate_bv &= hardware_xsave_mask;
```

6.  **寄存器清洗**：POSIX Server 在恢复 `ucontext_t` 时，强制屏蔽敏感的硬件标志位：

```
// 强制清洗 RFLAGS：保留算术标志，清除特权标志（如 IOPL, IF 等）
frame->rflags = (user_provided_rflags & 0x8D5) | 0x200;
frame->cs = USER_CS;     // 强行重置段寄存器，防止越权
frame->ss = USER_SS;
```

7.  POSIX Server 恢复信号掩码，调用 `sys_thread_set_state` 彻底恢复上下文。
8.  **AST 栈弹出**：POSIX Server 在完成现场恢复后，调用微内核接口弹出 AST 栈：

```
// 弹出 AST 栈
if (thread->ast_depth > 0) {
    thread->ast_depth--;
    thread->ast_stack[thread->ast_depth].sigreturn_cookie = 0;  // 清除 cookie
}
if (thread->ast_depth == 0) {
    thread->in_ast_checkpoint = false;
}
```

### 场景 F：SIGSTOP/SIGCONT 处理与 SA_NOCLDSTOP 检查

`SIGSTOP` 和 `SIGCONT` 是特权信号，采用**合作式挂起**机制，避免硬冻结导致的死锁。

**问题 1**：若直接调用 `sys_thread_suspend`，目标线程可能正处于 IPC 交互中途或持有跨进程共享锁，强行冻结会导致其他等待线程死锁。

**问题 2（SMP 逃逸竞态）**：CPU 0 设置了目标线程的 `stop_pending = true` 并发送 IPI，但在 IPI 到达之前，运行在 CPU 1 上的目标线程刚好越过了 AST 检查点返回用户态。线程会继续运行一段时间，导致 SIGSTOP 看起来"失效"。

**解决方案**：化"硬冻结"为"软拦截"，在 AST 边界安全挂起，并引入**确认握手机制**。

1.  **接收信号**：POSIX Server 收到 `SIGSTOP` 请求。
2.  **合作式挂起**：不直接调用 `sys_thread_suspend`，而是设置 `ast_pending = 1` 并打上 `STOP_PENDING` 标记，同时初始化停止确认计数器：

```
case SIGSTOP:
case SIGTSTP:
    // 合作式挂起：设置 AST 标记，等待线程到达安全边界
    proc->stop_pending_count = 0;  // 重置确认计数器
    proc->stop_generation++;       // 停止世代号（防止过期确认）
    
    for_each_thread_in_process(t, proc) {
        atomic_set(&t->ast_pending, 1);
        t->stop_pending = true;
        t->stop_generation = proc->stop_generation;  // 记录世代号
        // 若线程在其他 CPU 运行，发送 IPI 强制抢占
        if (t->state == THREAD_RUNNING && t->cpu != current_cpu)
            send_ipi(t->cpu, IPI_AST_KICK);
        proc->stop_pending_count++;  // 期望收到的确认数
    }
    proc->state = PROC_STOPPING;  // 标记为"正在停止"
    break;
```

3.  **AST 边界安全挂起**：线程到达 AST 检查点时，调度器发现 `STOP_PENDING`，此时线程必然不在执行敏感的内核级动作，安全挂起：

```
// 调度器出口 AST 检查
if (atomic_read(&thread->ast_pending)) {
    if (thread->stop_pending) {
        // 安全挂起点：线程即将返回用户态，无内核资源占用
        thread->state = TASK_STOPPED;
        thread->stop_pending = false;
        atomic_set(&thread->ast_pending, 0);
        // 发送确认 IPC（携带世代号防止过期确认）
        ipc_notify_async(POSIX_SERVER, MSG_THREAD_STOPPED, 
                         .tid = thread->tid,
                         .generation = thread->stop_generation);
        schedule();  // 切换到其他线程
    } else {
        // 常规信号投递路径
        ...
    }
}
```

4.  **确认等待**：POSIX Server 必须等待所有线程的 `MSG_THREAD_STOPPED` 确认，只有收到全部确认后才宣布进程已停止：

```
// POSIX Server 处理 MSG_THREAD_STOPPED
void handle_thread_stopped(pid_t pid, tid_t tid, uint64_t generation) {
    process_t *proc = lookup_process(pid);
    
    // 防御性检查：世代号不匹配则忽略（过期确认）
    if (generation != proc->stop_generation)
        return;
    
    proc->stop_pending_count--;
    
    // 所有线程都已确认停止
    if (proc->stop_pending_count == 0) {
        proc->state = PROC_STOPPED;
        
        // 检查 SA_NOCLDSTOP
        parent_sigchld_action = get_handler(proc->ppid, SIGCHLD);
        if (!(parent_sigchld_action.sa_flags & SA_NOCLDSTOP)) {
            deliver_sigchld_to_parent(proc->ppid, CLD_STOPPED);
        }
    }
}
```

5.  **SIGCONT 恢复**：收到 `SIGCONT` 时执行以下步骤：

```
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
- **规则 1**：`SIGCONT` 到达时，必须清除该进程所有待投递的停止信号（`SIGSTOP`/`SIGTSTP`/`SIGTTIN`/`SIGTTOU` 的 pending 位），即使这些信号已被屏蔽。
- **规则 2**：`SIGCONT` 本身即使被屏蔽，也必须唤醒已停止的进程。若注册了 `SIGCONT` Handler，恢复后再投递。

## 第四部分：十一道防弹级安全防线 (Bulletproof Defenses)

*   **防线 1：SIGKILL 内核原生保底与严格 SMP 同步**。`SIGKILL` 由微内核 Ring 0 直接拦截执行，不依赖 POSIX Server。即使 POSIX Server 死锁、OOM 或消息队列被塞满，管理员仍能杀掉恶意进程，确保系统"活性（Liveness）"。采用严格的 SMP 同步机制：获取所有 CPU 的 runqueue lock 后再检查线程状态和发送 IPI，防止线程在检查期间 migrate/preempt 导致漏发 IPI。确保所有 CPU 都脱离进程地址空间后才释放资源，防止 SMP CR3/TLB Tear-down Race 导致的三重故障。
*   **防线 2：SIGSTOP 合作式挂起与 SMP 竞态防护**。`SIGSTOP` 采用合作式挂起机制，在 AST 边界安全挂起，避免硬冻结导致的死锁。引入确认握手机制与世代号，确保所有线程真正停止后才宣布进程已停止，防止 SMP 逃逸竞态。
*   **防线 3：POSIX Server 自身免疫与最小权限原则**。
    - **异常隔离**：调用 `sys_register_posix_server`。其发生的异常不走常规路由，直接上报 Init 触发灾难恢复。
    - **Capability 细粒度拆分**：原始的 `CAP_PERM_WRITE_STATE` 权限过大（等价于 ptrace、arbitrary RIP write），拆分为 `CAP_PERM_SIGNAL_DELIVERY`、`CAP_PERM_EXCEPTION_RECOVER`、`CAP_PERM_SIGRETURN_RESTORE` 三个细粒度权限。
    - **RIP 白名单验证**：`sys_thread_set_state_safe` 强制验证 RIP 必须指向已注册的 handler、trampoline page 或已验证的 sigframe 区域。即使 POSIX Server 被 exploit，也无法注入任意 RIP 执行任意代码。
    - **防止全系统沦陷**：通过最小权限原则和 RIP 验证，确保 POSIX Server exploit 不会导致全系统 Ring 3 进程沦陷。
*   **防线 4：嵌套信号防护与可重入来源区分**。TCB 中维护固定深度（`MAX_AST_DEPTH = 8`）的 AST 栈，支持嵌套信号处理。每个栈帧记录 `AstContext` 类型（SIGNAL/SIGRETURN/EXCEPTION/TRAMPOLINE）和 `signo`，区分不同的可重入来源。检测 SIGSEGV 递归、异常发生在 sigreturn/trampoline 中等不可恢复状态，立即强杀进程，防止无限递归。
*   **防线 5：SROP 攻击防御与现场验证**。`SigStackFrame` 包含随机 Cookie 校验值，`sigreturn` 时校验与 AST 栈顶一致性。恢复现场前通过 `sys_thread_set_state_safe` 的 `RIP_POLICY_SIGFRAME` 验证 RIP Canonical 规范和用户空间范围，清洗 XSAVE 特性掩码。恢复时强制清洗 RFLAGS 特权位和段寄存器，防止伪造栈帧提权。
*   **防线 6：备用栈溢出防护与红区保护**。
    - **备用栈判断与溢出检查**：若 Handler 设置了 `SA_ONSTACK`，且线程有有效的备用栈，则使用备用栈。计算 `new_rsp` 时检查是否超出备用栈底部，溢出则放弃投递并强杀进程：
    
    ```
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
    
    - **红区扣除**：无备用栈时强制扣除红区：`new_rsp = (current_rsp - 128 - sizeof(SigStackFrame) - 8) & ~0xFULL;`。内存布局：`new_rsp` 处放 `__sigreturn` 地址（8 字节），`new_rsp + 8` 处放 `SigStackFrame`。
*   **防线 7：SIGCONT POSIX 强制语义**。`SIGCONT` 到达时清除所有停止信号 pending，即使被屏蔽也必须唤醒已停止进程。发送 `SIGCHLD` 前检查父进程 `SA_NOCLDSTOP` 标志。
*   **防线 8：续传令牌生命周期强绑定**。TCB 中记录 `current_restart_cookie`，任何新的系统调用（非 `sys_restart_syscall`）自动清零旧 Cookie。防止 `siglongjmp` 逃逸导致的混乱代理漏洞，避免旧 Cookie 被复用 fd 的恶意程序利用。
*   **防线 9：用户态协作安全点**。引入 `signal_defer_depth` 计数器和 `sys_signal_checkpoint` 系统调用，让运行时库（malloc、mutex、GC）主动声明 critical section。信号只在用户态声明的安全点注入，真正解决 async-signal-safe 问题，避免用户态锁重入死锁。
*   **防线 10：强制进入 Kernel Boundary 的多层机制**。采用 IPI 强制抢占（微秒级）、Timer Tick 检查（毫秒级兜底）、irq_work 延迟工作队列、TSX Abort 处理、Breakpoint 注入等多层机制，确保卡在用户态死循环、TSX transaction、用户态中断屏蔽等场景下的线程最终进入 kernel boundary，解决"不可达线程"问题。
*   **防线 11：sys_process_read_memory 的 COW 安全约束**。Core Dump 服务读取目标进程内存时，必须保证不破坏 Copy-on-Write 语义。使用 `get_user_pages_safe` 带 `GUP_FLAGS_READ_ONLY | GUP_FLAGS_NO_COW | GUP_FLAGS_NO_FAULT` 标志，确保只读访问、不触发 COW、不触发缺页。对于未映射的页填充 0，而不是分配新页。读取操作不修改目标进程的任何页表状态，确保 dump 本身不改变程序行为。

注意！代码已经申明 no-red-zone，且任务切换时会 FXSAVE (XSAVE) FXSTORE。

## 第五部分：复杂生命周期同步与协议

### 1. Fork 二阶段提交与冻结窗口期

1.  **Phase 1 (Freeze)**：微内核发 `MSG_FORK_BEGIN`。POSIX Server 置 `signal_freeze = true`。窗口期到达的信号一律丢入 `freeze_pending_queue`。返回快照版本 `V`。
2.  **Phase 2 (Copy & Resume)**：微内核发 `MSG_FORK_COMPLETE(pid, V)`。POSIX Server 初始化子进程，`signal_freeze = false`，最后 `drain_and_deliver()` 倾泻暂存队列补发信号。

### 2. 进程终止协议 (waitpid 与 SIGCHLD 竞态)

POSIX Server 内部充当了 Linux 中 `Task struct` 的部分角色，维护**僵尸进程表**：
1.  **子进程退出**：微内核发消息给 POSIX Server 告知退出码。
2.  **状态转移**：POSIX Server 将该进程转入 Zombie 状态，投递 `SIGCHLD`。
3.  **waitpid 消费**：父进程调用 `waitpid` 实则是向 POSIX Server 发送 IPC。若表中有 Zombie，立即出队返回；若无，父进程阻塞等待 IPC 回复。

---

## 第六部分：架构优缺点总结 (Pros and Cons)

### 🌟 核心优势 (The Pros)

1.  **顶级的调试生态 (Debugger-Friendly)**：异常即 IPC。调试器挂载异常端口，信号生成前即可完美拦截现场，告别 `ptrace`。
2.  **极致的内核安全**：微内核代码消灭了位图与蹦床。即便 POSIX Server 发生数组越界，系统内核绝不 Panic。
3.  **根治异步安全死锁 (Async-Signal-Safety Solved)**：信号仅在 AST 边界或明确系统调用处介入。告别"指令中途劈开"的噩梦，根除锁重入困扰。
4.  **准确的 I/O 恢复**：`SyscallRestartInfo` 配合驱动 Cookie，彻底解决非幂等调用被打断的数据重复问题。
5.  **最小权限原则与 RIP 验证**：Capability 细粒度拆分和 RIP 白名单验证，即使 POSIX Server 被 exploit，也无法注入任意 RIP 执行任意代码，防止全系统 Ring 3 进程沦陷。
6.  **COW 安全的内存读取**：`sys_process_read_memory` 保证不破坏 Copy-on-Write 语义，Core Dump 和调试器读取目标进程内存时不会改变程序行为，确保调试和取证的准确性。

### ⚠️ 工程挑战与缺点 (The Cons)

1.  **高昂的 IPC 性能开销**：一次 `#PF` 涉及四次内核/用户态切换（`肇事线程 -> 微核 -> POSIX Server -> 微核 -> Handler`）。极度依赖微内核 Fastpath 寄存器传参优化。
2.  **组件同步复杂度极高**：Token 失效、Fork 冻结窗口、端口超时降级必须严丝合缝，任意竞态都会导致 Use-After-Free 级别的逻辑漏洞。
3.  **驱动心智负担**：所有的阻塞驱动都必须维护一套机制，识别 `restart_cookie` 并提供正确的 `bytes_completed`，大幅提高了驱动开发的门槛。
4.  **RIP 验证与白名单维护开销**：微内核需要维护 handler 地址白名单，每次 `sigaction` 系统调用都需要更新白名单。这增加了内核与 POSIX Server 之间的同步复杂度，但这是防止 POSIX Server exploit 导致全系统沦陷的必要代价。
5.  **get_user_pages 实现复杂度**：`sys_process_read_memory` 需要实现类似 Linux `get_user_pages` 的复杂机制，正确处理 COW、缺页、页表遍历等场景。这增加了微内核的代码复杂度，但这是保证 Core Dump 不破坏程序行为的必要代价。

