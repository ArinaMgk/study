
# 程序

- `prehost\atx-x86-flap32\atx-x86-flap32.loader.cpp`
	- 用于加载静态例程和活动内核并转移控制权

- `prehost\atx-x86-flap32\atx-x86-flap32.cpp`
	- 静态例程和活动内核，包含以下模块

- ↑ 向上滚屏
- ↓ 向下滚屏


## ISSUE

- [ ] 20250706 ISSUE Each time subapp (Ring3) print %d or other integer by outsfmt() will panic, but OutInteger() or Kernel Ring0 is OK.
- [ ] 「在裸机C++中，在纯虚函数的子类实现中使用 可变参数 会产生错误。」

## 模块

按照流行处理器结构、参考主流操作系统进行设计。

以下均限于 x86

### 默认初始例程な中断と异常 `handler.cpp`

- `void Handint_PIT()`
	- PIT 中断
	- 影响：位置闪烁和任务切换

- `void Handint_RTC()`
	- RTC 中断
	- 影响：位置闪烁

- `void Handint_KBD()`
	- 键盘中断
	- 影响：回显字符

### 内存管理模块 `memoman.cpp`

十分简单的内存分配机

```c++
#define mem_area_exten_beg 0x00100000
class Memory {
// area-basic: 0x7E00 .. 0x80000
// area-exten: mem_area_exten_beg .. mem_area_exten_beg + areax_size
public:
	// MCCA 的两段内存区域
	static byte* p_basic;// 必满
	static byte* p_ext;// 可选
	static usize areax_size;
public:
	static void clear_bss();

	// 向上对齐4k
	static usize align_basic_4k();

	// 统计物理内存大小
	_TEMP static usize evaluate_size();

	// 分配物理内存
	static void* physical_allocate(usize siz);

	// 返回内存大小的可读字符串，例如"64K"
	static rostr text_memavail(uni::String& ker_buf);
};

Paging kernel_paging;
extern stduint tmp;
// 初始化分页机制
_TEMP void page_init();

// 接口
extern void(*(*_physical_allocate)(stduint size));
```

#### 分页模块

### 系统调用模块 `syscall.cpp`

```c++
extern "C" void call_gate();
extern "C" void* call_gate_entry();
void syscall(syscall_t callid, stduint paracnt = 0, ...);
```

### 进程管理模块 `taskman.cpp`

目前假设了两个驻留内存的程序。

本模块包含线程的管理。

`word TaskRegister(void* entry)` 注册两个程序

### 平台特殊模块

- 分段模块
	- 统一平坦分段，更多研究请看历史Commit。

### 依赖库模块

接口
- cpuid 获取CPU信息
- ioport 端口接口
- manage CPU管理指令接口

文件系统与文件格式
- ELF

设备
- PIT
- keyboard
- rtc clock
- harddisk
- i8259A

系统
- task
- interrupt
- paging

通用
- mcore
- conformat consio debug stream string ...

