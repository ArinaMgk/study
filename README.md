
# 总结よUNISYM

编程语言
- 裸机编程
- 逻辑原理
	- 调用约定、寄存器约定
- 包管理
- 内存分配
- 多线程

Rust
- unsafe 像 C#
- enum 像 D
	- 适合SysMessage
- struct spread operator 在 fork() 中大可能会用到

Rust
- 圆括号可以省，大括号不可以省（M/C++相反）
- 指针运算不隐含，指针不能直接进行大小比较（除非它们指向同一个分配对象的内部）
- Rust Trait 机制比C++接口的优势
	- 可以针对已有类型 `impl T for C<t>`
	- 在Trait中实现相当于不纯み虚函数（非纯虚函数的默认实现）


#  Topic

### 核心系统与虚拟化 (OS & Virtualization)

- Linux Kernel 核心子系统开发（内存管理、调度器、VFS）
- Microkernel / 形式化验证微内核 (seL4, Zircon, Mach)
- Unikernel (OSv, MirageOS)
- Hypervisor / VMM (KVM, Xen, Intel VT-x, AMD-V)
- eBPF / XDP (内核级高性能追踪与网络过滤)
- 容器底层运行时 (runC, Kata Containers)
- 实时操作系统 (RTOS) 与硬实时调度 (FreeRTOS, VxWorks, RT-Thread)
    

### 编译技术与工具链 (Compiler & Toolchain)

HEREPIC
(OUT) Kasha
HAL for MCU/MPU
Magice/COTLAB

- LLVM IR 优化与后端生成 (LLVM Backend)
- GCC 内部机制 (GIMPLE / RTL)
- 链接器开发与优化 (mold, lld, GNU ld)
- JIT (Just-In-Time) 编译器引擎开发 (V8, LuaJIT)
- Debugger 底层实现 (GDB / LLDB / ptrace)
- 二进制翻译 / 动态插桩 (QEMU, Rosetta 2, DynamoRIO, Frida)
- C++ 运行时库 / 标准库实现 (libc, libc++, libstdc++)
    

### 极致性能优化与高性能计算 (Performance & HPC)

- CPU 微架构级指令优化 / 循环展开
- SIMD / 向量化编程 (AVX-512, ARM Neon, RISC-V V-Extension)
- 硬件性能计数器分析 (PMU, Linux `perf`, Intel VTune)
- 高性能内存分配器 (jemalloc, tcmalloc, mimalloc)
- 无锁 / 互斥锁无关数据结构 (Lock-free / Wait-free Data Structures)
- Cache 一致性协议深度分析 (MESI, MOESI)
- GPU 异构计算与底层调优 (CUDA, ROCm, OpenCL)
- 量化/高频交易 (HFT) 超低延迟系统开发
    

### 固件与硬件接口 (Firmware & Hardware Interface)

- 现代 UEFI / EDK2 固件开发
- 开源固件替代方案 (Coreboot, Libreboot)
- 嵌入式引导程序 (U-Boot, Barebox)
- 高级配置与电源接口 (ACPI) 表工程
- 服务器基板管理控制器 (BMC / OpenBMC)
- PCIe / NVMe 协议栈与驱动开发
- 现代总线控制器驱动 (XHCI, Thunderbolt)
    

### 体系结构与芯片设计 (Architecture & IC Design)

- RTL 逻辑设计 (Verilog, SystemVerilog, VHDL)
- 现代硬件敏捷开发 (Chisel, SpinalHDL)
- 指令集架构 (ISA) 深度定制 (RISC-V 扩展设计)
- SoC 片上总线架构 (AMBA AXI, AHB, APB)
- 计算机体系结构模拟器 (gem5, Verilator)
- NPU / 张量处理单元 / AI 芯片架构设计
- EDA (电子设计自动化) 核心算法与工具开发
- FPGA 原型验证与综合
    

### 网络与存储基础设施 (Storage & Network Infra)

- 用户态网络协议栈 (DPDK, VPP)
- RDMA (远程直接内存访问) 与 InfiniBand
- 用户态文件系统 (FUSE)
- 分布式存储引擎底层 (LSM-Tree, B+ Tree)
- 高性能 KV 数据库底层架构
- 零拷贝技术 (Zero-copy)
    

### 逆向工程与底层安全 (Reverse Engineering & Security)

- 二进制静态分析与逆向 (IDA Pro, Ghidra)
- 内核级漏洞挖掘与 Fuzzing (AFL++, syzkaller)
- 侧信道攻击缓解与防御 (Spectre / Meltdown 级)
- 信任执行环境 (TEE) 架构 (Intel SGX, ARM TrustZone)
- 内存安全语言系统编程 (Rust for Linux)

### 计算机图形学与渲染引擎 (Computer Graphics & Rendering)

- 现代底层图形 API 开发 (Vulkan, DirectX 12, Metal)
    
- 传统图形接口与封装 (OpenGL, WebGPU)
    
- 着色器编程与渲染管线定制 (GLSL, HLSL, SPIR-V 编译)
    
- 光线追踪算法与底层硬件加速 (Ray Tracing, DXR, OptiX)
    
- 纯 CPU 软渲染器开发 (Software Rasterizer)
    
- 3D 游戏引擎底层开发 (自研内存分配器、多线程渲染系统)
    
- 物理引擎开发 (碰撞检测、刚体/流体动力学计算, 类似 PhysX/Havok)
    
- 几何处理与网格算法 (Mesh Processing, 细分曲面)
    

### UI 框架与人机交互底层 (GUI Frameworks & HCI)

- Qt / QML 底层源码级定制、裁剪与性能调优
    
- LVGL 等嵌入式轻量级 GUI 移植与显存/2D硬件加速优化
    
- 即时模式图形界面开发 (Immediate Mode GUI, 如 Dear ImGui)
    
- 跨平台 2D 矢量渲染后端开发 (Skia, Cairo, ANGLE)
    
- 显示服务器与视窗管理系统底层 (Wayland Compositor, X11, SurfaceFlinger)
    
- 字体排版与渲染引擎底层 (FreeType, HarfBuzz)
    

### 音视频处理与流媒体 (Audio/Video & Multimedia)

- 音视频编解码器底层算法实现与优化 (H.265/HEVC, AV1, AAC)
    
- 多媒体框架架构深度定制与硬件加速 (FFmpeg, GStreamer)
    
- 流媒体通信底层协议栈与 C++ 源码级调优 (WebRTC)
    
- 数字信号处理 (DSP) 与音频降噪、回声消除 (AEC) 算法工程化
    
- 操作系统音频子系统与驱动层开发 (ALSA, PulseAudio, PipeWire)
    

### 机器人与工业自动化 (Robotics & Industrial Control)

- 机器人操作系统底层通信机制与实时性改造 (ROS / ROS2 DDS)
    
- 工业现场总线协议栈开发 (EtherCAT, PROFINET, Modbus)
    
- 无人机/自动驾驶飞控系统固件开发 (PX4, ArduPilot)
    
- 电机控制底层算法实现 (FOC, 空间矢量 PWM, PID)
    
- SLAM (同步定位与建图) 算法的 C++ 底层工程化与 SIMD 加速
    
- 可编程逻辑控制器 (PLC) 运行环境底层开发 (IEC 61131-3)
    

### AI 基建与算力系统 (AI Infrastructure & Compute)

- AI 编译器开发与计算图优化 (MLIR, TVM, XLA)
    
- 深度学习推理引擎底层开发 (TensorRT, ONNX Runtime, OpenVINO)
    
- CUDA C++ / Triton 自定义算子 (Kernel) 编写与显存/寄存器极致调优
    
- 大模型分布式训练框架底层通信库开发 (NCCL, MPI)
    

### 汽车电子与智能网联 (Automotive Software)

- AUTOSAR (Classic & Adaptive) 基础软件层 (BSW) 及微控制器抽象层 (MCAL) 开发
    
- 车载以太网与高可靠中间件协议栈 (SOME/IP, 车载 DDS)
    
- 智能座舱实时系统/虚拟机适配 (QNX 裁剪, 车载 Hypervisor)
    
- CAN / CAN FD / LIN 汽车电子总线控制器驱动开发
    

### 边缘计算与物联网底层 (Edge Computing & IoT)

- 物联网极简操作系统内核开发 (Zephyr, LiteOS, mbedOS)
    
- 轻量级网络协议栈深度裁剪与移植 (LwIP, MQTT, CoAP)
    
- 低功耗广域网 (LPWAN) 底层协议栈 (LoRaWAN, NB-IoT, BLE)

