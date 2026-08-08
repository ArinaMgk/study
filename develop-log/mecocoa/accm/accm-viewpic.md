
# 20260602

# BMP 图片显示程序实施计划

## Summary

目标是先做出一个可运行的 BMP 图片查看程序，只支持第一版最小可用范围：读取本地 `.bmp` 文件，解码为内存像素，显示到 Mecocoa 的窗口里，并能正常关闭。第一阶段优先打通“文件 -> 像素 -> 窗口”链路，不先接入完整的 `picture.hpp` 图像系统抽象。

默认采用“先做最小 BMP 解码器，再做最小 viewer”的路线，复用 `subapps/test.cpp` 的建窗与事件循环模式，参考 `cube.cpp` / `game_breakout.cpp` 的逐像素 framebuffer 写入方式。

## Step By Step

### Step 1. 锁定第一版 BMP 支持范围

这一阶段只做格式范围定义，不写复杂抽象。

要做的事：
- 明确只支持 Windows BMP 的 `BM` 文件头。
- 明确只支持 `BITMAPFILEHEADER + BITMAPINFOHEADER`。
- 明确只支持 `biCompression == BI_RGB`。
- 明确只支持 `24-bit` 和 `32-bit` BMP。
- 明确不支持调色板 BMP、RLE 压缩、bitfields、动画、多帧。

这一阶段的产出：
- 一个实现边界明确的第一版规格，避免后面在解码器里不断加条件分支。
- 后续 `BMP.h` 只需要为这个范围定义结构和常量。

默认规则：
- `24-bit` 输入按 `BGR` 读入。
- `32-bit` 输入按 `BGRA` 或 `BGRX` 处理。
- `biHeight > 0` 视为 bottom-up。
- `biHeight < 0` 视为 top-down。

### Step 2. 设计 BMP 模块的最小接口

这一阶段只决定 `BMP.h/.cpp` 对外提供什么能力，不把它做成完整图像框架。

要做的事：
- 在 `BMP.h` 定义 BMP 文件头结构、信息头结构、压缩常量、行对齐辅助。
- 设计一个最小 C/C++ 友好的解码接口，职责仅限“从 BMP 文件读出像素和尺寸信息”。
- 约定输出格式直接是 `uni::Color*` 线性缓冲，避免第一版引入多像素格式转换框架。
- 约定返回给调用方的信息至少包含：宽、高、像素缓冲、缓冲大小。

推荐接口方向：
- 输入：文件路径或文件描述符。
- 输出：`width`、`height`、`uni::Color* pixels`。
- 由 BMP 模块分配像素内存，调用方负责释放。

这一阶段的产出：
- `BMP.h` 中最小但可落地的接口声明。
- 生命周期和 ownership 规则清楚，不和 `picture.hpp` 混杂。

默认规则：
- 第一版不暴露 `ImageBuffer`。
- 第一版不实现 `IImageCodec`。
- 第一版只服务 `viewpic`。

### Step 3. 实现 BMP 文件头解析与合法性校验

这一阶段在 `BMP.cpp` 中完成“能不能读”判断。

要做的事：
- 打开文件并读取 BMP 文件头和信息头。
- 校验文件长度是否至少覆盖头部。
- 校验签名是否为 `BM`。
- 校验 `biSize` 是否满足 `BITMAPINFOHEADER` 大小。
- 校验 `planes == 1`。
- 校验 `bitCount` 仅允许 `24` 或 `32`。
- 校验 `compression == BI_RGB`。
- 校验像素偏移和图像尺寸是否合理，防止越界读取。

失败时要返回明确错误：
- 文件打不开。
- 不是 BMP。
- 位深不支持。
- 压缩方式不支持。
- 文件损坏或尺寸非法。
- 内存分配失败。

这一阶段的产出：
- 一条稳定的“读头 -> 验证 -> 提取关键信息”的路径。
- 后续像素解码只处理已通过验证的输入。

### Step 4. 实现像素解码并转成 `uni::Color`

这一阶段完成真正的 BMP 解码。

要做的事：
- 根据 `bitCount` 分流处理 `24-bit` 和 `32-bit`。
- 正确计算 BMP 每行 4 字节对齐的 stride。
- 处理 bottom-up 和 top-down 两种行顺序。
- 为输出缓冲分配 `width * height` 个 `uni::Color`。
- 把 BMP 像素转换为窗口可直接使用的 `uni::Color`。
- `24-bit` 时补全 alpha 为 `0xFF`。
- `32-bit` 时保留 alpha 或默认设为 `0xFF`，实现上保持简单一致。

推荐转换规则：
- BMP `BGR` -> `Color { b, g, r, a=0xFF }`
- BMP `BGRA` -> `Color { b, g, r, a }`

这一阶段的产出：
- 一个可被 `viewpic` 直接消费的 ARGB 风格线性像素缓冲。
- 不依赖 `LayerManager`、`GraphicForm`、`IImageSurface`。

### Step 5. 实现 `viewpic` 的最小窗口程序

这一阶段在 `viewpic.cpp` 中把应用跑起来。

要做的事：
- 参考 `subapps/test.cpp` 的程序骨架。
- 接收命令行路径参数。
- 调用 BMP 解码接口读取图片。
- 根据图片尺寸创建窗口。
- 分配窗口 framebuffer，或直接使用图片尺寸对应的显示缓冲。
- 调用 `sys_create_form`、`sys_set_form_buffer`、`sys_update_form`。
- 实现关闭按钮和 `Alt+F4` 退出。
- 出错时输出简单错误信息并退出。

推荐窗口策略：
- 客户区大小默认等于图片大小。
- 窗口总大小 = 客户区 + 边框标题栏。
- 第一版不做滚动和缩放。
- 图片过大时先允许大窗口，不先实现缩放策略。

这一阶段的产出：
- 一个最小可运行的 `viewpic`。
- 对单张 BMP 可直接打开并显示。

### Step 6. 实现“图片像素 -> 窗口缓冲”的拷贝逻辑

这一阶段只处理显示，不处理格式解析。

要做的事：
- 将解码后的 `uni::Color*` 图片缓冲逐行复制到窗口 framebuffer。
- 保证索引方式统一为 `y * width + x`。
- 若窗口大小与图片大小一致，则直接整块逐像素拷贝。
- 若后续要预留更大窗口，本阶段只做左上角放置，不做缩放。

这一阶段的产出：
- 图像真正出现在窗口上。
- 显示层和解码层职责分离。

默认规则：
- 第一版不做居中。
- 第一版不做背景填充。
- 第一版不做透明混合。

### Step 7. 做最小错误处理和资源回收

这一阶段保证程序不是“一次性 demo”。

要做的事：
- 处理命令行缺参数。
- 处理文件打不开。
- 处理 BMP 解码失败。
- 处理窗口创建失败。
- 处理内存申请失败。
- 程序退出时释放图片缓冲和窗口缓冲。
- 正常关闭 form。

这一阶段的产出：
- 一个可重复运行、不会明显泄漏资源的 viewer。
- 基本故障场景下有可理解输出。

### Step 8. 跑通后再考虑回接 `picture.hpp`

这一阶段不是第一版实现的一部分，只是后续收敛方向。

要做的事：
- 把 BMP 解码结果映射到 `ImageBuffer`。
- 再决定是否实现 `IImageCodec` 的 BMP 版本。
- 再决定是否为 `IImageSystem::LoadImage()` 提供 BMP 后端。
- 确保当前 `viewpic` 已可工作后再做抽象回收。

默认规则：
- 不在第一版 viewer 里强行引入 `IImageSystem`。
- 不在第一版 BMP 模块里实现 metadata、surface、document。

## Important Interfaces

第一版需要新增或确定的对外接口行为：

- `BMP.h`
  - BMP 头结构定义
  - BMP 压缩常量定义
  - 最小解码函数声明
  - 输出 ownership 规则声明

- `BMP.cpp`
  - 完成“文件 -> `uni::Color*`”的最小解码实现

- `viewpic.cpp`
  - 完成“参数 -> 解码 -> 建窗 -> 拷贝 -> 刷新 -> 事件循环”的应用主流程

接口原则：
- 第一版接口面向“能显示 BMP”，不是面向“完整图像框架”。
- 输出类型优先适配现有 Mecocoa framebuffer，而不是追求抽象通用性。

## Test Plan

至少验证以下场景：

- 正常打开一个 24-bit、无压缩、bottom-up BMP。
- 正常打开一个 32-bit、无压缩 BMP。
- 校验颜色通道正确，避免红蓝颠倒。
- 校验图像上下方向正确，避免倒置。
- 校验每行 4 字节对齐正确，避免花屏或错行。
- 传入不存在的文件路径，程序能报错退出。
- 传入非 BMP 文件，程序能报错退出。
- 传入暂不支持的 BMP 变体，程序能明确报“不支持”。
- 点击关闭按钮能退出。
- `Alt+F4` 能退出。

## Assumptions

- 第一版只支持本地文件读取，不支持流式输入。
- 第一版只做 BMP 解码，不做 BMP 编码。
- 第一版只做显示，不做缩放、滚动、拖动、平移。
- 第一版不接入 `picture.hpp` 的 `IImageCodec` / `IImageSystem`。
- 第一版输出像素格式直接使用 `uni::Color` 兼容当前窗口缓冲。
- 第一版优先复用 `subapps/test.cpp` 的 form-buffer 模式，而不是先接 `GraphicForm` 高层封装。

