
ArceOS是一个用Rust编写的实验性模块化操作系统（Unikernel），受到Unikraft的启发。

### 2. 主要目录结构

#### 2.1 arceos/ - ArceOS内核源码

这是项目的核心目录，包含ArceOS内核的源代码，与主线版本有所差异，是经过剪裁的版本以便学习。

**核心模块（modules/）：**

- `axalloc` - 内存分配器模块
- `alt_axalloc` - 替代内存分配器
- `axconfig` - 配置管理
- `axdisplay` - 显示驱动
- `axdriver` - 设备驱动框架
- `axfs` - 文件系统
- `axhal` - 硬件抽象层
- `axlog` - 日志系统
- `axmm` - 内存管理
- `axnet` - 网络栈
- `axruntime` - 运行时
- `axsync` - 同步原语
- `axtask` - 任务调度
- `bump_allocator` - Bump分配器
- `riscv_vcpu` - RISC-V虚拟CPU支持

**API层（api/）：**

- `arceos_api` - ArceOS原生API
- `arceos_posix_api` - POSIX兼容API
- `axfeat` - 功能特性管理

**用户库（ulib/）：**

- `axstd` - Rust标准库替代
- `axlibc` - C标准库实现

**示例和教程：**

- `examples/` - 示例程序（helloworld, httpclient, httpserver, shell等）
- `tour/` - 教程示例（u_1_0到u_8_0, m_1_0到m_3_1, h_1_0到h_4_0）
- `exercises/` - 练习题（print_with_color, support_hashmap, alt_alloc, sys_map, simple_hv, ramfs_rename）

**平台支持（platforms/）：**

- `riscv64-qemu-virt.toml` - RISC-V QEMU虚拟机
- `aarch64-qemu-virt.toml` - ARM64 QEMU虚拟机
- `x86_64-qemu-q35.toml` - x86_64 QEMU虚拟机
- `aarch64-raspi4.toml` - 树莓派4
- `x86_64-pc-oslab.toml` - x86_64物理机
- `aarch64-bsta1000b.toml` - 特定ARM开发板

#### 2.2 challenges/ - 挑战题目

包含挑战题目说明和资源文件，如lab1.md描述的内存分配器优化挑战。

#### 2.3 course/ - 教学资料

包含第三阶段的课程PPT文件（stage3-1.pptx到stage3-9.pptx）。

#### 2.4 scripts/ - 评测脚本

包含自动化测试脚本：

- `total-test.sh` - 执行所有测试
- `test-print.sh` - 测试带颜色打印功能
- `test-ramfs_rename.sh` - 测试文件系统重命名
- `test-alt_alloc.sh` - 测试替代分配器
- `test-support_hashmap.sh` - 测试哈希表支持
- `test-sys_map.sh` - 测试系统映射
- `test-simple_hv.sh` - 测试简单虚拟化
- `tour_test.sh` - 教程测试

#### 2.5 crates/ - 依赖的Rust crate

包含手动修改的模块，如kernel_guard。

#### 2.6 tour_books/ - 教程文档

包含教程相关的文档和图片。

### 3. 构建和运行系统

#### 3.1 构建系统

- 使用Makefile作为主要构建系统
- 支持多架构（x86_64, riscv64, aarch64）
- 支持多平台（QEMU虚拟机、树莓派4等）
- 支持多种配置选项（SMP、日志级别、设备支持等）





