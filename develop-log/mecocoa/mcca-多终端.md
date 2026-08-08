



# 20260714

```
在 x86 no GUI 模式下，默认有4个屏幕，即 Bcons[TTY_NUMBER]
初始状态，只有 Bcons 0 启动并绑定了 cot 程序。
我现在想：
1、Bcons 0 的 cot 退出时，自动重新启动一份 cot
2、【Lazy Create】使用 F2 F3 F4 能切换到 Bcons 1 2 3，在切换到 Bcons i 的时候，如果 Bcons i 没有绑定 cot，就启动一份 cot 并绑定上
3、定义了 ProcessBlock* Bcons_pcot[TTY_NUMBER] = {};，辅助实现以上功能
---
不要改动 Taskman::Exit，统一再按下 F1~4的时候判断吧
---

```

# 20260430

```
现在内核GUI只有一个固定的Shell，这限制了多终端的实现。我现在想实现多终端窗体。以下是我的方案：
1、ProcessBlock 现在有一个 Dnode* focus_tty = nullptr;指向当前IO绑定的TTY。focus_tty默认为0，表示不接受控制台输入和没有输出。
2、程序fork/exet时子进程可以继承focus_tty。
3、程序创建(exec)子进程默认没有绑定tty。如果是窗口程序不影响什么，如果是控制台程序，则无法进行IO，对此一般需要「先申请Shell进程（包含一个TTY），再在创建进程的时候绑定上」
4、一个进程管理一个Shell，是为了通过关闭按钮关闭Shell窗口，更好地管理资源。这样，鼠标和键盘消息直接转发到Shell进程，Shell再操作TTY的缓冲区，类似PTY (Pseudo-Terminal)。Shell的关闭按钮按下时，会先尝试关闭绑定的进程，最后再关闭Shell进程。
5、（vtty_type_t添加一个Vector对象）为每个Shell添加一个 vector 进程组 (Process Group)，直接存储 PID，指示当前TTY绑定了哪些进程。关闭 Shell 时，向该终端关联的所有进程组发送终止信号。（从尾端开始关闭）这样 vector的 Count 就是引用计数，用于管理Shell进程的引用计数。当引用计数为0时，关闭Shell进程。在 fork 等操作时，子进程不仅要继承指针，还要增加该 Dnode 的引用计数（如果你的 VFS 有引用计数机制的话），以防止父进程退出后，终端设备被意外销毁。

请依据现在我的代码，评估我的方案，并告诉我是否可行。Implementation Plan 请使用中文编写。
---
先不要修改代码。有几点需要注意
- 销毁Shell的时候不要遗漏 vector 等资源的清理
- 考虑 Exet 应该怎么处理
- pgroup 改名为 proc_group
---
对于3. Shell 进程启动时由父线程或者fileman负责创建窗体（这些窗体需要有所记录，待Shell进程退出则自动销毁），这一点就不需要操心了，不需要添加syscall或者操作码。
---
好。现在x86的初始TTY是在bool Consman::Initialize()的使用 CreateVconsole创建的。我们为了通用性，将if (_TEMP 1)这个代码块删掉，在 void serv_file_loop() 中 #ifdef _ARC_x86 CreateFile 这部分同时创建TTY和init程序，让我们的x86变得更通用。
---
现在有init 程序，在D:\her\COTLAB\src\cotlab.cpp。但是init不是shell！shell的职责是创建窗口，转发输入输出，可以绑定init控制台程序，也可以绑定其他任何的程序。现在init 没有问题，shell不能依赖于Init。
你理解错了。不是全局一个shell进程，而是每个shell窗体一个shell进程。你可以把shell的代码直接放在内核。
---
你问Shell应该挂载什么程序？这不是他的职责！fileman创建shell并挂载init，exec创建shell并挂载它传入参数的程序，决定权肯定不在于serv_shell_process，你不能在Shell里面写死任何的程序。此后再修改代码之前必须经过我的同意。
[现在的 Shell 进程一启动，就会立刻进入 sysrecv 阻塞挂起状态，它收到消息后，才会去创建 CreateVconsole 分配窗体和 TTY，随后将传入的 ProcessBlock 挂载到该 TTY 上，并且把它 Append 进就绪队列。如果 p 运行结束，Shell 自动清理窗体并自我销毁。]
别乱找了，这个根本不是问题。
问题出在用户程序通过tty IO的系统调用上。serv_shell_process创建的tty虽然正常返回，但是可能有问题。程序在执行sysc_OUTC的时候崩溃了。我十分怀疑 (pid->focus_tty)->offs 指向的不是一个真正的 Console_t 地址，请检查给 vttys 节点offs赋值等位置有没有问题。
---
serv_shell_process这些操作CreateVconsole等GUI的函数的好像非常吃栈空间。我把程序栈从 4K 改到接近 64K才能正常运行，请检查是什么原因（不要修改代码）
```

```
【vaddr=0xF000FF53】我发现问题了，我内存中 0x0000 地方存的数组就是PF的位置。这说明可能是 vtable或者指针 的问题。请检查哪里没有 Nullptr保护。
当前代码在摘除的时候是否考虑到了「如果 A->C, B->C，A退出时 不仅要释放A现在阻塞发送/接收的进程，还要把C的队列指针指向B，而不是直接断开导致B从此与C失联？」
现在没有问题。看来问题出在delete pfrm;中。我们应该怎么分析，先分析它的解析构造函数吗？
```


