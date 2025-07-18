メココア




20250718
- 放弃方案「Syscall参数位置调用门自带」
- 针对退格的处理
- Shift实现大写字母键入
- 小键盘等显示所有键入
- 全键盘支持

> TTY处理字符接口 应该在用户层

20250717
- remove TaskFlatRegister, TaskFlat_t
- ProcessBlock
- Revive Bochs
- Stage
- 疑似中断处理有问题，以下为未来解决方法
	- 不使用默认的 TSS 进程切换
	- 使用纯汇编 包围中断处理程式，重新设计中断处理结构


```c++
	GIC.enAble();
	// GIC.enAble(false);
	// GIC.enAble(false);
	// CallFar(0, 8 * 9);
/*
开中断 R1 会崩溃（开过即使随后关了也不行）
关中断正常使用 uni.Console （在R1代码中）

将R1提升为R0也能正常使用 uni.Console
使用系统调用的方法不会出问题
因此あ推断问题可能在中断里

---

2108: 发现调用 tty 的其他方法都会崩溃
最终总结：
- 提升为R0无异常
- 与调用类的静态成员无关

懂了懂了，与这些都没有关系，我把 IO 指令禁用了都(IOPL)！！！——2127
*/
...
	tty.auto_incbegaddr = 0;
	// tty.setStartLine(--tty.crtline + tty.topline);
	++tty.crtline;
	stduint jjj = (tty.crtline + tty.topline) * tty.area_total.x;
	outpb(CRT_CR_AR, CRT_CDR_StartAddressHigh);
	outpb(CRT_CR_DR, 0 >> 8);
	outpb(CRT_CR_AR, CRT_CDR_StartAddressLow);
	outpb(CRT_CR_DR, 0 & 0xFF);// Here IOPL=0 but Ring=1
}
else if (keycode == 0x51 && tty.crtline < tty.area_total.y - tty.area_show.height) { // PgDn
	tty.auto_incbegaddr = 0;
	tty.setStartLine(++tty.crtline + tty.topline);
}
...
```


20250710~20250716
- NEU,Shenyang chore-work
- Read OS-Principle's Video

20250709

优化TTY输入方案：Kbd -> KbdBridge -> Distrib Crt TTY (+ Use its parser(scan->filtered_show) get getc)  -> Wait for Focus App Getc()
- 切换 APP 时暂时保留内容

OSD 「在裸机C++中，在纯虚函数的子类实现中使用 可变参数 会产生错误。」

20250708

设想TTY输入方案：Kbd (+ Global Parser) -> Interf-Buf -> Distrib Crt TTY -> Distrib Focus App (+ Use its parser get getc) -> Wait for App Getc()

20250707
- 废弃的想法
	- mecocoa top banner(单独TTY,隐式光标支持)，同时实现完整的键盘解析
- Commit
- Progress
	- TTY0~3 & 激活 【暂未实现绑定】
		- ESC返回动态内核(含banner任务), KernelTTY=TTY0
		- F1 subappa, TTY1
		- F2 subappb, TTY2
		- F3 TTY3 
		

20250706 SYSCALL 返回值、标识当前进程
- ISSUE Each time subapp (Ring3) print %d or other integer by outsfmt() will panic, but OutInteger() or Kernel Ring0 is OK.
- 开始看王道的计算机操作系统考研课（理论派的）
- 操作系统专业课 由广大操作系统开发者、学习者、使用者总结的概念, 很多概念没有形成共识

20250705 TTY0控制台显示初步(光标不一定非要在屏幕中)
Issue
- VMware 不识别上下等方向键

20250704 无阶段性进度
- 现有基础
	- consio
	- class Console_t : public OstreamTrait, public IstreamTrait;
	- class HostConsole
- BareConsole 控制台：列数、总行数、显示行数、顶行、光标(XY)

>20250704 stroke her: [控制台键盘处理策略]用于解析到文本缓冲区，联系键盘驱动响应快捷键和LED等
>一个程序有且只有一个本命IO(由TSS管理)、默认重定向控制台TTY0 (IstreamTr. OstreamTr.)

20250703 配环境; 使用int指令进行系统调用而不是通过调用门; 使用寄存器传递参数;

20250702 配环境

20250701 重启计划
