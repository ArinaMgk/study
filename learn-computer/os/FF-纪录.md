20250706 SYSCALL 返回值、标识当前进程
- ISSUE Each time subapp (Ring3) print %d or other integer by outsfmt() will panic, but OutInteger() or Kernel Ring0 is OK.

20250705 TTY0控制台显示初步(光标不一定非要在屏幕中)
Issue
- VMware 不识别上下等方向键

20250704 无阶段性进度
- 现有基础
	- consio
	- class Console_t : public OstreamTrait, public IstreamTrait;
	- class HostConsole
- BareConsole 控制台：列数、总行数、显示行数、顶行、光标(XY)

20250703 配环境; 使用int指令进行系统调用而不是通过调用门; 使用寄存器传递参数;

20250702 配环境

20250701 重启计划
