
```c++
		//{TEMP} UEFI GUI specific implementation
		if (ch == '\b') {
			if (ptext_1) {
				auto str = ptext_1->text.reflect();
				if (*str) {
					ptext_1->text[-1] = 0;
					ptext_1->text.Refresh();
				}
			}
		}
		else if (ptext_1 && ch) ptext_1->text << ch;
		if (ptext_1) ptext_1->doshow(0);
/*

	// syscall(syscall_t::OUTC, (usize)"Ohayou\n\r", 8);

	if (1) {
		// Memory::pagebmap->dump_avail_memory();
		// mempool.dump_available();
		// ploginfo("[Console] There are %[u] layers, f=%[x], l=%[x]", global_layman.Count(), global_layman.subf, global_layman.subl);
	}

*/

	// [demo] window
	if (1) {
		Rectangle rect{ Point(200, 40), Size2(160, 80) };
		// Label
		auto plabel = new uni::witch::control::Label("QwQ~");
		plabel->sheet_area = Rectangle(Point(0, 0), Size2(8 * 5, 16));
		plabel->doshow(0);
		plabel_1 = plabel;

		form0.Title = "Ciallo~>v<";// new (&form0.Title) String((char*)form0_title_text, sizeof(form0_title_text));
		form0.AppendControl(plabel);
		form0.setSheet(global_layman, rect, (Color*)mem.allocate(rect.getArea() * sizeof(Color)));
		global_layman.Append(&form0);
	}
	if (1) {
		Rectangle rect{ Point(400, 40), Size2(160, 80) };
		auto ptext = new uni::witch::control::TextBox();
		ptext->sheet_area = Rectangle(Point(2, 2), Size2(8 * 18, 25));
		ptext->doshow(0);
		ptext_1 = ptext;

		form1.Title = "Test TextBox";
		form1.AppendControl(ptext);
		form1.setSheet(global_layman, rect, (Color*)mem.allocate(rect.getArea() * sizeof(Color)));
		form1.setFocus(ptext);
		global_layman.Append(&form1);
		ptext->Start();
	}
```

# 20260601

```
现在内核：D:\her\unisym\inc\cpp\Witch这些图形库只是被内核使用的，其实可以被用户程序使用，我在想能不能作为用户GUI使用，请评估我的这一想法（不要change code）

方案 ：客户端渲染与共享缓冲区

程序在自己的用户内存空间中分配一个像素缓冲区（Color 数组），并基于 VideoControlInterfaceMARGB8888 实例化 LayerManager。
Witch 的所有控件渲染（绘制按钮、文本框、窗体等）完全在用户态中完成，直接写入这个局部缓冲区。
渲染完成后，用户态程序通过系统调用或 IPC（如共享内存）将缓冲区（或脏矩形区域 dirty_area）发送给内核的窗口合成器（Compositor），内核只负责将其“贴”到屏幕对应的位置。

这样需要每隔一段时间统计有没有图形变化，如果有就通知内核。这些接口大部分已经实现，参考 

```

|接口类别|作用|内核侧职责|用户侧职责|现状判断|
|---|---|---|---|---|
|窗口/宿主接口|给 Witch 一个可存在的窗体宿主|创建、销毁、移动、激活窗口；维护 Z 序、焦点、关闭行为|请求创建/关闭窗口，响应宿主状态变化|已有雏形，FNEW/FDEL 基本在做这件事|
|像素缓冲区接口|提供 Witch 的绘制目标|接受用户缓冲区或共享面；校验格式、尺寸、权限|分配 Color* 缓冲区，交给 Witch 绘制|已有雏形，FBID 已能绑定用户缓冲区|
|像素格式接口|保证两边对颜色布局理解一致|定义并检查 MARGB8888 等格式|按约定格式输出像素|基本已有，当前明显偏向 VideoControlInterfaceMARGB8888|
|提交/刷新接口|把用户态绘制结果交给内核合成|接收刷新请求并加入合成/脏区流程|绘制完成后提交整块或脏区|已有 FUPD，但现在语义偏“整块复制”，脏区还没真正吃透|
|脏区接口|避免全量拷贝和全量重绘|合并 dirty rect，调度局部合成|记录哪些区域变了并上报|框架已有，dirty_area 已存在，但用户窗体路径还没完全打通|
|输入事件接口|把鼠标键盘交给 Witch|采集输入、命中窗口、投递事件|从消息队列取事件，交给应用逻辑/控件逻辑|已有雏形，Form::onrupt() + FMSG 已在工作|
|消息队列接口|在宿主和应用之间传递 GUI 消息|为窗体维护事件队列，支持阻塞/唤醒|拉取消息并处理|已有，FMSG 很接近可用形态|
|定时器接口|支持闪烁光标、动画、延迟动作|提供计时源并投递 onTimer|注册/取消定时器，消费定时消息|已有，FTIM 已具备基础能力|
|合成接口|把窗口内容贴到屏幕|负责桌面合成、遮挡、最终输出|不直接碰屏幕，只提交内容|已有，内核的 LayerManager2/Compositor 路径已在做|
|共享内存接口|降低复制成本|建立共享页/共享表面句柄并管生命周期|使用共享缓冲区而不是裸用户指针|目前还没有完整形态，现状更像“用户地址 + 内核拷贝”|
|安全/生命周期接口|避免悬挂指针和越界|校验窗体归属、内存范围、进程退出清理|正确释放窗口和缓冲区|需要加强，尤其是长期保存用户缓冲区指针这块|
|资源接口|字体、主题、光标等共享资源|提供系统资源或默认实现|选择使用系统资源或自带资源|部分已有，但还不像统一接口|

如果 Witch 要被别的宿主环境复用，最好的方式不是要求别人适配整个系统，而是要求别人实现一组“宿主接口”。Witch 本体只依赖这些接口，就能跑在内核、用户程序、仿真器、测试框架，甚至别的 OS 上。

下面这张表里的接口名称是我建议的命名，不是你现有代码已经固定的名字。

| 接口名称                   | 作用             | 使用者需要实现什么                     | 必须性   | 备注                               |
| ---------------------- | -------------- | ----------------------------- | ----- | -------------------------------- |
| IWitchSurface          | 提供绘制目标         | 提供像素缓冲区、宽高、stride、像素格式        | 必须    | 这是最基础接口，没有它就没法画                  |
| IWitchPresenter        | 把画好的内容提交到宿主    | 提供 present() 或 presentDirty() | 必须    | 可以是拷贝、共享内存、贴屏、上传 GPU             |
| IWitchInputSource      | 向 Witch 提供输入事件 | 鼠标、键盘、焦点、窗口事件注入               | 必须    | 没有它只能做静态 UI                      |
| IWitchTimerSource      | 提供定时能力         | 注册、取消、触发定时器                   | 高     | 文本框光标闪烁、动画、延时行为会依赖它              |
| IWitchMessageQueue     | 让应用取到 GUI 消息   | 支持投递、取出、阻塞等待                  | 高     | 如果 Witch 采用消息驱动模型，这个非常重要         |
| IWitchWindowHost       | 提供窗口级宿主能力      | 创建、关闭、移动、调整大小、激活窗口            | 高     | 单窗口嵌入场景可弱化，多窗口时基本必需              |
| IWitchClipboard        | 剪贴板支持          | 读写文本或二进制数据                    | 可选    | 文本框、编辑器类控件会想用到                   |
| IWitchTextRenderer     | 字体与文本绘制        | 测量文本、栅格化文本、绘制字形               | 高     | 若 Witch 自带位图字库可降为可选              |
| IWitchImageCodec       | 图片解码/编码        | PNG/BMP/JPEG 等资源读取            | 可选    | 只有图像控件或主题资源需要                    |
| IWitchCursorHost       | 鼠标光标形态控制       | 箭头、文本光标、拖动态等                  | 可选    | 没有也能跑，只是体验差                      |
| IWitchThemeProvider    | 主题和系统配色        | 颜色、边框、标题栏、默认字体等               | 可选    | 可先内置默认主题                         |
| IWitchResourceLoader   | 加载字体、图片、布局资源   | 文件、内存块、打包资源读取                 | 可选    | 对移植很方便                           |
| IWitchAllocator        | 自定义内存管理        | alloc/free/realloc            | 可选    | 裸机、内核、受限环境里很有价值                  |
| IWitchLogger           | 诊断输出           | 日志级别、输出目标                     | 可选    | 调试和移植期很有帮助                       |
| IWitchCompositor       | 多层合成           | 管理多个 layer/sheet 的混合输出        | 视模式而定 | 如果 Witch 只画单面板，可不需要；做桌面/窗口系统时很重要 |
| IWitchSharedBufferHost | 共享缓冲区协作        | 创建、映射、同步共享表面                  | 可选    | 只有跨进程/跨特权级时才重要                   |

**按“绝对最小可运行集”看**  
如果别人只是想把 Witch 跑起来，哪怕只在一个单窗口里显示，最少需要这几个：

| 接口名称               | 必须性             |
| ------------------ | --------------- |
| IWitchSurface      | 必须              |
| IWitchPresenter    | 必须              |
| IWitchInputSource  | 必须              |
| IWitchTextRenderer | 必须或由 Witch 内建替代 |
| IWitchTimerSource  | 高               |
| IWitchMessageQueue | 高               |

**按不同接入场景看**

|场景|最少需要的接口|
|---|---|
|静态界面截图渲染|IWitchSurface|
|单窗口交互程序|IWitchSurface IWitchPresenter IWitchInputSource IWitchTimerSource IWitchMessageQueue|
|多窗口桌面程序|上述全部 + IWitchWindowHost|
|跨进程 GUI|上述全部 + IWitchSharedBufferHost|
|主题化完整 GUI 框架|上述全部 + IWitchThemeProvider IWitchClipboard IWitchResourceLoader|

**我建议的一组更具体的方法名**  
下面这组名字比较适合作为第一版接口草案：

| 接口                     | 建议方法名                                                                                      |
| ---------------------- | ------------------------------------------------------------------------------------------ |
| IWitchSurface          | lockPixels() unlockPixels() getWidth() getHeight() getStride() getPixelFormat()            |
| IWitchPresenter        | present() presentRect(const Rectangle&) presentRects(const Rectangle*, size_t)             |
| IWitchInputSource      | pushMouseMove() pushMouseButton() pushKeyEvent() pushFocusEvent()                          |
| IWitchTimerSource      | startTimer() stopTimer() resetTimer()                                                      |
| IWitchMessageQueue     | postMessage() peekMessage() getMessage() waitMessage()                                     |
| IWitchWindowHost       | createWindow() destroyWindow() moveWindow() resizeWindow() setWindowTitle() requestClose() |
| IWitchTextRenderer     | measureText() drawText() getLineHeight()                                                   |
| IWitchSharedBufferHost | createSharedSurface() mapSharedSurface() unmapSharedSurface() flushSharedSurface()         |

**如果你想把接口压得更薄**  
甚至可以只保留 3 个总接口，把别的能力挂进去：

|总接口|包含内容|
|---|---|
|IWitchHostSurface|像素面、present、dirty rect|
|IWitchHostEvent|输入、消息队列、定时器|
|IWitchHostWindow|窗口、焦点、关闭、标题|

这样接入方实现起来更直接，但后期可扩展性稍弱。

**我的建议**  
如果你的目标是让别人容易接入 Witch，我建议优先公开这 6 个名字：

- IWitchSurface
- IWitchPresenter
- IWitchInputSource
- IWitchTimerSource
- IWitchMessageQueue
- IWitchWindowHost

```
谢谢，我先继续实现吧。如果遇到问题再来提问。如果现有框架解决不好有些问题，那就是我们开始重写的时机了
```

18:01

这条判断我很赞同。

先继续实现，拿真实问题来逼出边界，通常比现在就大动架构更值。当你碰到的问题开始反复出现，而且明显不是单点修补能解决的时候，那基本就是该重写或至少重分层的信号了。

到时候可以重点看这几类迹象：

- 同一种逻辑在内核态和用户态不断重复写
- 为了接一个新控件/新程序，要到处打补丁
- 消息、缓冲区、定时器的语义越来越不清晰
- 性能问题不是局部优化能解决，而是接口本身限制住了
- 生命周期和所有权开始频繁出错

这些一旦集中出现，重写就不是“折腾”，而是在止损。

你现在先把文本查看器推进下去是很好的选择。等你遇到卡点，直接把现象、预期和相关代码位置拿来，我们可以一起判断这是“补接口”还是“该重构了”。

---





