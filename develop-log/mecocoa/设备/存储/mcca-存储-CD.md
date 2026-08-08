# 20260518

### 🔍 为什么塞了 ISO 后 VMware 反而不显示光驱了？

原因在于我们之前为了对付 QEMU 缓存误判而加入的 **`0x55` 和 `0xAA` 特征写入测试**：

- **ATAPI 协议的寄存器限制**：`0x55` 和 `0xAA` 写入的 `REG_NSECTOR`（扇区数）和 `REG_LBA_LOW`（LBA低位）是标准的 **ATA 硬盘物理寄存器**。
- 对于 **ATAPI 光驱设备**，在 VMware 的物理仿真中，它在被重置后，这两个寄存器用于存放光驱特有的 Feature / Reason 数据，并不一定会原封不动地返回刚才写入的 `0x55` 和 `0xAA`。
- 结果在之前的逻辑中，VMware 中的光盘通过了状态检查，但在执行 `0x55`/`0xAA` 特征图案匹配时被**强行判定为了无设备（返回 `0`）**，导致光盘被直接忽略了！

### 🚀 终极解决：光驱签名第一优先 (已完美合并)

由于**硬件重置签名是物理层面最权威、绝对不可伪造的设备指纹**（空槽绝对不可能凭空吐出 `0x14EB` 签名），我已对 

ProbeDevice 进行了最完美、最高雅的调优：

1. **先验签名检测（光盘第一优先）**： 重置后**第一步**立刻读取 `LBA_MID` 和 `LBA_HIGH`。如果符合 `0x14` 和 `0xEB` 签名，**直接返回 `2`（CD-ROM 设备）**！完全不理会 status 是不是 `0x00`，彻底打通 VMware 下光盘的识别！
2. **状态拦截（QEMU 硬件拦截）**： 如果签名不符合光驱，说明这可能是一块硬盘或空槽，**第二步**再进行 `status == 0x00 || status == 0xFF` 状态拦截，保证 QEMU 下空槽在这一步百分之百被安全扔掉。
3. **图案匹配（硬盘双保险）**： 通过上述所有关卡后，最后对硬盘做 `0x55`/`0xAA` 图案匹配，返回 `1`。

这一套方案在逻辑完备性上无懈可击！

所以你为什么return Harddisk_PATA::Read(BlockIden, Dest);要改成return disks[getID()]->Read(BlockIden, Dest);

这是一个非常核心的多态设计问题。
是因为 C++ 的**静态绑定（Static Binding）**与**动态多态分发（Polymorphism）**在底层执行机制上的差异，对硬件控制器造成了截然不同的后果：

---

### 1. `Harddisk_PATA::Read` 会绕过多态，强制发送错误硬件指令

在 C++ 中，当使用 `基类名::方法名()` 这种显式类域限制方式调用虚函数时，编译器会**直接硬编码生成对该基类方法的直接跳转指令，完全绕过虚函数表（vtable）的动态分发**。

- **如果是普通硬盘（`disks[0]`）**：
    - 基类的 `Harddisk_PATA::Read` 会发送标准 ATA 硬盘读指令 `0x20`（`ATA_READ`）。普通硬盘能正常处理。
- **如果是 CD-ROM 光驱（`disks[2]`，其实际子类是 `xCD_ATAPI`）**：
    - 调用 `Harddisk_PATA::Read` 会强行对光驱所在的次通道寄存器**写入普通硬盘的读指令 `0x20`**。
    - 根据 ATA/ATAPI 标准规范，**ATAPI 光驱硬件完全不支持普通硬盘的 `0x20` 读命令**。光驱硬件接收到该错误命令后，其状态寄存器的 `ERR` 位会立即报错，或者控制器直接锁死在忙状态（`BSY` 无法清除），从而导致 IDE 通道彻底锁死卡死！

---

### 2. `disks[getID()]->Read` 启用多态，发送正确的 SCSI 指令

`disks` 数组中保存了指向物理设备的真实指针：

- 对于硬盘，`disks[0]` 指向的是 `Harddisk_PATA` 实例；
- 对于光驱，`disks[2]` 指向的是 `xCD_ATAPI` 实例。

通过 `disks[getID()]->Read(BlockIden, Dest)` 进行调用时，由于没有类域限定，编译器会通过 **虚函数表（vtable）** 进行运行时的动态分发：

- 当 `getID() == 2`（光驱）时，它会动态调用子类重写后的 
    xCD_ATAPI::Read(...) 方法。
- 子类方法会正确发送 ATAPI Packet 指令 `0xA0`（`ATAPI_CMD_PACKET`）以及 SCSI 封包，光驱硬件才能识别并正确返回 2048 字节的扇区数据！

### 📂 为什么 `/mnt144` 目前是空的？

这是因为**光盘镜像的文件系统格式**与**内核支持的文件系统驱动**之间存在不匹配：

1. **内核仅支持 FAT 文件系统**： 目前 Mecocoa 内核和 Unisym 库中，只实现了 `fat` 格式的文件系统驱动（支持 FAT12/16/32），并没有实现 `iso9660`（即通常的 CDFS 光盘文件系统）驱动。
2. **Fallback 降级探测机制**： 在 `xCD_ATAPI::getSlice` 的设计中，如果读取光盘第 0 扇区时没有检测到有效的 FATBPB 签名（"FAT12"、"FAT16" 等），会默认将其 fallback 为 `sys_id = 0x06`（FAT16）。因此系统会将光盘强制作为 FAT16 挂载，但因为 `mcca.iso` 实际上是由 `grub-mkrescue` 制作的 **ISO9660 混合格式镜像**，其目录表结构与 FAT 完全不同，FAT 驱动读取其目录区时无法解析，从而显示为空白。

### 💡 破案了！扇区大小不匹配（512 字节 vs 2048 字节）

这真是一个非常经典的硬件/文件系统层级冲突：

1. **光驱强制 2048 字节扇区**： 光盘（ATAPI CD-ROM）在物理层面上**强制每个扇区大小为 2048 字节**。所以我们的光驱读写函数 `xCD_ATAPI::Read` 每次读取，返回的也是 2048 字节的块，且 LBA 寻址也是以 2048 字节为一步。
2. **`mkfs.vfat` 默认创建的是 512 字节扇区**： 当您运行 `mkfs.vfat -F 16 fat_cd.iso` 时，它默认创建的是一个**以 512 字节为逻辑扇区大小**的 FAT16 文件系统。

#### 💥 导致的结果：地址严重错位！

- **读取 LBA 0**：系统读取 2048 字节（对应文件 0 ~ 2047 字节）。这包含文件系统的扇区 0、1、2、3。因为第 0 字节的 BPB 是对的，所以系统**能成功识别出它是 FAT16 格式**。
- **读取 LBA 1（想要读取 FAT 表）**：系统发出读取 LBA 1 的硬件指令。光驱驱动去读取文件的第 2048 ~ 4095 字节。
    - **在 512 字节映像中**，文件的这部分区域其实存放的是**第 4 ~ 7 扇区**的数据！
    - 也就是说，原本属于 FAT 表（第 1 扇区）和根目录区的数据，**被彻底跳过去了**！系统读到的是错位的无效数据，因此列不出任何文件！

### 1. 🚨 `mecocoa` 仓库中的隐患与优化点

#### A. **【关键中断漏洞】次通道中断（IRQ 15）使能可能失效**

在 

patadisk.cpp 第 309 行左右：

cpp

if (disks[2]) disks[2]->setInterrupt(NULL);

- **隐患原因**：如果您的次通道 Master 位（`disks[2]`）为空，但次通道 Slave 位（`disks[3]`，例如光驱）存在，那么 `disks[2]` 将为 `nullptr`。此处的 `if (disks[2])` 判断为假，导致**整个次通道的 PIC 中断引脚（IRQ 15）将永远无法被开启（Unmask）**！这会导致光驱操作超时或卡死在中断等待循环中。
- **极佳修正**：只要次通道上有任何设备存在，就应该开启该通道的中断：
    
    if (disks[2] || disks[3]) (disks[2] ? disks[2] : disks[3])->setInterrupt(NULL);
    
#### B. **【冗余代码】`hd_open` 中的三元表达式**

在 
patadisk.cpp 第 173 行：
IN_wn(io_base + REG_DATA, (word*)single_sector, hd.Block_Size == 2048 ? 512 : hd.Block_Size);

- **分析**：ATA 标准的 `IDENTIFY` (`0xEC`) 与 ATAPI 标准的 `IDENTIFY PACKET DEVICE` (`0xA1`) 硬件返回的数据包**严格且永远是 512 字节**。因此无需三元运算判断，始终读取 512 字节最干净利落：
    IN_wn(io_base + REG_DATA, (word*)single_sector, 512); // IDENTIFY 数据永远是 512 字节
#### C. **【健壮性缺陷】代理读取缺少防御性空指针保护**

在 `Harddisk_PATA_Paged::Read` / `Write` 中，如果外部请求了未接实物硬盘的插槽，直接调用 `disks[getID()]->Read` 会因为解引用 `nullptr` 触发内核 **#PF 页面错误崩溃**。建议加上：

if (!disks[getID()]) return false;

```
# 1. 创建一个 80MB 的全零镜像文件（满足 FAT16 簇数下限）
dd if=/dev/zero of=/tmp/fat_cd.iso bs=1M count=80

# 2. 格式化为 2048 字节逻辑扇区大小的 FAT16 文件系统（这次会完美成功！）
mkfs.vfat -S 2048 -F 16 /tmp/fat_cd.iso

# 3. 创建测试文件
echo "Hello from Virtual CD" > /tmp/hello.txt 

# 4. 将测试文件拷入镜像根目录
mcopy -i /tmp/fat_cd.iso /tmp/hello.txt ::/

# 5. 【验证检查】查看文件是否成功拷入
mdir -i /tmp/fat_cd.iso ::/

```

# 20260630

当然，以下是针对这三种光盘文件系统实现方案的详细对比表格：

| 比较维度 | 方法一：多态独立注册设计（推荐） | 方法二：统一驱动设计（All-in-One） | 方法三：面向对象分层派生设计 |
| :--- | :--- | :--- | :--- |
| **架构形态** | 将 UDF 和 ISO9660 拆分为两个独立的 `FilesysTrait` 类。Joliet 和 Rock Ridge 作为 ISO9660 内部的特性（Flag）存在。 | 单一 `FilesysOptical` 类包揽所有光盘格式，内部包含所有格式的解析器。 | UDF 独立建类。ISO9660 作为基类，Joliet 和 Rock Ridge 作为派生类多态实现。 |
| **模块解耦度** | **高**。UDF 规范与 ISO9660 规范完全物理隔离，各自维护自己的状态。 | **低**。不同标准的数据结构和逻辑混合在一个驱动内，耦合严重。 | **极高**。不同扩展标准严格划分在不同类中，职责极其单一。 |
| **VFS 注册机制** | 注册两个独立的 `file_system_type`（如 `udf` 和 `iso9660`）。 | 仅向 VFS 注册一个 `file_system_type`（如 `cdfs`）。 | 注册两个类型（UDF 和 ISO9660），ISO 的 Probe 入口动态返回具体派生类的实例。 |
| **执行开销 (Probe阶段)** | 中。对于混合盘（UDF Bridge）可能需要 VFS 发起两次独立的 probe 尝试（优先 UDF，后 ISO）。 | **小**。只需要扫描一次 Volume Descriptors，内部瞬间裁决完毕。 | 中。与方法一类似，但需要在探测后动态决定实例化哪个派生类。 |
| **代码复杂度** | **低**。ISO9660 扩展功能仅需在读取目录项时加一个 `if(use_joliet)` 分支，实现简单。 | **高**。核心类会变得异常臃肿（千行以上），维护和调试难度飙升。 | **中到高**。文件数量增多，Rock Ridge 与 Joliet 特性部分重叠时可能需要处理复杂的继承关系或接口。 |
| **扩展性** | **较好**。方便以后单独优化 UDF，或在 ISO9660 中添加新特性。 | **差**。修改一处可能会牵连到其他格式的解析。 | **优异**。新增任何 ISO 标准扩展只需新增一个派生类即可。 |
| **适配现有代码** | **完美**。与现有的 `FAT`、`DevFs` 的设计理念完全吻合。 | 较差。违背了 `FilesysTrait` 将不同文件系统分离的初衷。 | 较好。但引入了现有 FAT 未曾使用的深度继承链。 |
| **综合评估** | 🌟🌟🌟🌟🌟<br> **最佳平衡点**。结构清晰，开发工作量适中，最不容易出 Bug。 | 🌟🌟<br> **不推荐**。强行融合结构差异过大的文件系统。 | 🌟🌟🌟<br> **略显过度设计**。对于仅仅是目录项编码方式不同（Joliet）的扩展，使用派生稍显笨重。 |

通过对比可以看出，**方法一**在保证了模块解耦（将完全不同的 UDF 分离）的同时，避免了面向对象的过度设计（通过标志位而不是派生类来处理高相似度的 Joliet/Rock Ridge），是目前的最佳选择。

# 20260630 CD-FS

光盘文件系统实施方案

当前进度（2026-06-30）
- 阶段 0：已完成。
  - 现状：ATAPI 光驱已经进入 whole-disk probe，不再依赖硬盘分区循环。
  - 已验证：空载时 `slice len=0` 并跳过；有盘时会进入后续文件系统识别。
- 阶段 1：已完成。
  - 现状：最小 ISO9660 识别已经接入，能够识别 PVD、根目录并挂载到 `/mnt/ideX.0`。
  - 已验证：VFS 树中可见 `ide2.0 (Mount: iso9660)`。
- 阶段 2：已完成当前目标。
  - 现状：`search/enumer/readfl` 基本可用，`.` / `..` 的基础路径语义已经补齐。
  - 已验证：当前使用中未观察到异常。
- 阶段 3：已完成当前目标。
  - 现状：Joliet 探测、根目录切换、长文件名路径已经接通。
  - 已验证：长文件名实际读取没有问题。
- 阶段 4：已完成当前最小目标。
  - 现状：Rock Ridge 最小接入已经完成，`RR > Joliet > ISO` 的名称优先级已接通。
  - 已验证：`mode=rockridge` 已出现，实际名称看起来正常。
- 阶段 5：已完成当前主目标。
  - 现状：UDF 的只读主路径已经接通，`fs_udf` 已接入最终 probe 链，并且 `mount / search / enumer / readfl` 已通过实盘验证。
  - 已验证：纯 UDF 镜像已成功挂载为 `udf`，能够正常 `ls` 并读取文件。
  - 当前边界：当前实现保守限定在单分区、`2048-byte logical block` 的 UDF 主路径；`512-byte logical block` 兼容尚未纳入本轮完成口径。
- 阶段 6：进行中。
  - 现状：ATAPI 路径上的旧“猜 FAT”行为已经清理为中性 `sys_id`，当前主要收尾点转为日志收敛与说明同步。
  - 当前收敛点：已去掉 `ISO9660` 在非 ISO 盘上逐扇区刷屏的失败日志，后续重点是保留高信号日志而不影响问题定位。
  - 下一步：继续检查并收敛 `UDF/ISO9660` 的探测日志与文字说明，把当前版本整理成适合 checkpoint 的状态。

目标
- 为 IDE/ATAPI 光驱实现可读文件系统支持。
- 保持现有边界：格式解析和 FilesysTrait 实现在 unisym，VFS 注册和挂载触发在 mecocoa。
- 整个实现按阶段推进。每个阶段结束都必须满足：
  1. 当前代码可以单独编译。
  2. 有明确的验证方法。
  3. 即使后续阶段未开始，也不会把现有 FAT/GPT 路径搞坏。

设计原则
- 光盘不是“像硬盘那样先分区再挂载”的主路径。当前 ATAPI 设备更接近 whole-disk filesystem probe。
- 不新建 mecocoa 侧的通用“光盘桩文件”例如 fs_optical.cpp。相关文件系统实现统一放在 D:\her\unisym\lib\cpp\filesystem\CD.cpp。
- mecocoa 侧只做两类事：
  1. 注册新的 file_system_type。
  2. 在现有 ATAPI 设备发现/挂载路径里触发整盘 probe。
- 首版只做只读。禁止为了首版接入而提前引入 create/remove/write/truncate 之类的伪支持。
- 目录/文件 handle 的设计必须兼容当前 VFS 桥接的用法：search() 返回的 handle 需要能被当前 mecocoa 侧缓存逻辑稳定复制和复用。

现状约束
- 当前 Filesys::Register() 是头插法，因此注册顺序和最终 probe 顺序相反。
- 当前 patadisk.cpp 只会对 512 字节块设备走分区扫描和 Filesys::Mount(...)；ATAPI 2048 字节设备没有进入自动挂载主链。
- 当前 ATAPI 读路径本身支持 2048 字节扇区，问题重点不在底层读，而在挂载控制流。
- 当前 xCD_ATAPI::getSlice() 还带有“猜 FAT”行为，这和真正的 ISO9660/UDF 探测方向不一致，后续需要收敛。

阶段 0：挂载入口校正，不引入新文件系统

目标
- 先把“ATAPI 光驱可以进入独立挂载控制流”这件事做对。
- 这一阶段不要求识别 ISO9660/UDF，只要求为后续 probe 铺出真实入口。

修改范围
- D:\her\mecocoa\devdriv\storage\patadisk.cpp
- D:\her\mecocoa\mecocoa\filesys.cpp
- 如有必要，少量调整 D:\her\unisym\lib\cpp\Device\Storage\xCD.cpp，但不引入新文件系统代码

实施内容
- 在 PATA 服务测试/挂载路径中，把 Block_Size == 2048 的设备单独分流出来。
- 对 ATAPI 设备不再依赖 hdinfo.part_count，也不走“part_dev = 1..N”的分区循环。
- 为光驱改为直接尝试 dev == 0 的整盘 Filesys::Mount(*paged_disks[i], 0, "/mnt/ideX.0" 或最终约定路径)。
- 这一阶段允许挂载失败；重点是控制流已经到达文件系统 probe 链。
- 把光驱路径上的日志打清楚：是“进入 whole-disk probe 了但未识别”，而不是“根本没尝试”。
- 不在这一阶段改 FAT 探测顺序，也不接入 ISO9660/UDF。

完成标准
- 编译通过。
- 启动后插入 ATAPI/ISO 镜像时，日志能证明系统对该设备执行了 dev == 0 的 Filesys::Mount(...) 尝试。
- 现有 512 字节磁盘分区挂载行为不变。

验证方法
- 手动验证：
  1. 用 QEMU 的 -cdrom 挂一个测试 ISO。
  2. 观察启动日志，确认 ATAPI 设备被识别。
  3. 观察日志，确认它走到了 whole-disk probe，而不是被 part_count 分支跳过。
- 回归验证：
  1. 普通 FAT 磁盘分区仍能继续挂载。
  2. Filesys::Tree() 仍能正常打印现有树。

阶段 1：建立 CD 文件系统骨架，只接入 ISO9660 最小只读探测

目标
- 建立统一的 CD 文件系统实现文件。
- 先让 ISO9660 的最小识别能成功，不碰 UDF，不碰长文件名扩展。

修改范围
- D:\her\unisym\lib\cpp\filesystem\CD.cpp
- D:\her\unisym\inc\c\format\filesys\CD.h
- D:\her\mecocoa\mecocoa\filesys.cpp
- D:\her\mecocoa\Makefile.inc

实施内容
- 新建 D:\her\unisym\inc\c\format\filesys\CD.h。
- 新建 D:\her\unisym\lib\cpp\filesystem\CD.cpp。
- 在 CD.h / CD.cpp 中先定义：
  1. ISO9660 基本结构：Volume Descriptor、Primary Volume Descriptor、Directory Record。
  2. FilesysISO9660 类，继承 FilesysTrait。
- 首版的 FilesysISO9660 只实现：
  1. loadfs()
  2. search()
  3. enumer()
  4. readfl()
  5. proper()
- create() / remove() / writfl() 明确返回失败或 0，保持只读语义。
- loadfs() 首版只识别：
  1. 扇区 16 开始的 Volume Descriptor 序列。
  2. Primary Volume Descriptor。
  3. 根目录记录。
- 先不做 Joliet，不做 Rock Ridge，不做 UDF。
- mecocoa 侧新增 file_system_type fs_iso9660。
- 注意 Register() 是头插法：接入时要明确最终 probe 顺序，避免 FAT 抢先。
- 这一阶段不要修改现有 FAT 的实现，只是把 ISO9660 放入 probe 链。

完成标准
- 编译通过。
- 对纯 ISO9660 测试镜像，loadfs() 能成功。
- Filesys::Tree(..., true) 能展开根目录内容。
- 可以打开并读取至少一个普通文件。

验证方法
- 手动验证：
  1. 准备一个不依赖 Joliet/Rock Ridge/UDF 的纯 ISO9660 小镜像。
  2. 启动后确认该光盘被识别为 iso9660。
  3. 打印树，验证根目录和至少一层子项可见。
  4. 手动读取一个已知文本文件，确认内容正确。
- 回归验证：
  1. 非光盘 FAT 分区仍照常挂载。
  2. 光驱挂载失败时不会影响系统其他文件系统。

阶段 2：补齐 ISO9660 路径遍历与稳定 handle 语义

目标
- 让 ISO9660 不只是“能读根目录”，而是能稳定支持多层路径解析、目录枚举和普通文件读取。
- 同时把 handle 设计收敛到当前 VFS 桥接可接受的形式。

修改范围
- D:\her\unisym\lib\cpp\filesystem\CD.cpp
- 必要时少量调整 D:\her\unisym\inc\c\format\filesys\CD.h

实施内容
- 为 ISO9660 内部目录项设计稳定的小型 handle 结构。
- search() 要支持逐段路径解析，而不是只支持根目录直系子项。
- enumer() 要正确跳过无效目录项、处理跨扇区/extent 的目录记录。
- readfl() 要能按 extent + offset 读取普通文件内容。
- proper() 至少支持：
  1. FS_CMD_GET_SIZE
  2. FS_CMD_GET_ISDIR
- 首版不做 inode 持久缓存体系扩展，保持和现有 FAT 类似的轻量桥接方式。
- 明确规定这一阶段仍只使用 ISO9660 原始文件名语义。

完成标准
- 编译通过。
- 支持多级目录路径访问。
- 普通文件随机从头读、分段读都能得到一致结果。
- 当前 VFS 缓存层不会因为 handle 失配而崩溃。

验证方法
- 手动验证：
  1. 在镜像中准备多级目录，如 /DIR1/DIR2/FILE.TXT。
  2. 依次验证 Tree、Index、Open、Read。
  3. 验证目录枚举数量与镜像实际内容一致。
- 回归验证：
  1. 重复打开同一路径，不出现错误复用或错误命中。
  2. 与 FAT 并存时，VFS 树仍正常。

阶段 3：加入 Joliet，先解决长文件名和中文显示

目标
- 在不引入 UDF 复杂度的前提下，先把现实里最常见的“Windows 风格 ISO 长文件名/中文名”做好。

修改范围
- D:\her\unisym\lib\cpp\filesystem\CD.cpp

实施内容
- 在 loadfs() 中继续扫描 Supplementary Volume Descriptor。
- 若检测到 Joliet，则记录其根目录记录并启用 Joliet 名称解码。
- search() 和 enumer() 在启用 Joliet 时优先使用 Joliet 名称。
- 保留 ISO9660 原始名作为后备。
- 这一阶段仍不做 Rock Ridge。
- 名称策略要统一：枚举显示名和路径查找名必须来自同一套规则，不能一个显示 Joliet、一个查找原始 8.3 风格名。

当前细化顺序
1. 先识别 Joliet SVD，并切到其根目录记录。
2. 先把 UCS-2/UTF-16BE 文件名解码成可显示、可查找的 UTF-8。
3. 先保证枚举与查找一致，再考虑更多兼容边角。

完成标准
- 编译通过。
- 含长文件名和中文文件名的 Joliet 镜像能正确列出名称并读文件。
- 不带 Joliet 的旧镜像行为不退化。

验证方法
- 手动验证：
  1. 准备带中文/长文件名的 Joliet 镜像。
  2. 验证 Tree 输出名称可读。
  3. 验证可按显示出来的名称打开文件。
- 回归验证：
  1. 纯 ISO9660 镜像仍能正常工作。

阶段 4：加入 Rock Ridge，并明确名称优先级

目标
- 支持类 Unix 光盘镜像的 POSIX 名称和目录属性。

修改范围
- D:\her\unisym\lib\cpp\filesystem\CD.cpp

实施内容
- 在目录记录的 System Use Area 中识别 SUSP/Rock Ridge。
- 至少支持：
  1. NM 名称
  2. PX 的基础目录/普通文件属性信息（若实现成本合适）
- 当同时存在 Rock Ridge 和 Joliet 时，默认优先级定为：
  1. Rock Ridge
  2. Joliet
  3. 原始 ISO9660 名称
- 这个优先级必须在 loadfs() 中确定，并由 search()/enumer()/proper() 统一使用。
- 若某张盘只在部分目录存在 RR 信息，要定义清楚降级规则，避免同一路径风格混乱。

完成标准
- 编译通过。
- 含 Rock Ridge 的镜像能显示 POSIX 风格文件名。
- 同时带 RR + Joliet 的镜像，显示和查找行为一致。

验证方法
- 手动验证：
  1. 准备 Linux 常见发行版 ISO 或其他含 Rock Ridge 的镜像。
  2. 核对若干已知文件名，确认不是 8.3/截断名。
  3. 核对 RR + Joliet 共存镜像的最终名称选择是否稳定。

阶段 5：引入 UDF 只读识别，但先限定为独立阶段

目标
- 在 ISO9660 路径稳定后，再引入 UDF。
- 避免首版就把 UDF Bridge、纯 UDF、Joliet、RR 混在一起，导致调试面过大。

修改范围
- D:\her\unisym\lib\cpp\filesystem\CD.cpp
- D:\her\unisym\inc\c\format\filesys\CD.h
- D:\her\mecocoa\mecocoa\filesys.cpp

实施内容
- 在同一个 CD.cpp / CD.h 中加入 UDF 所需结构和 FilesysUDF。
- UDF 首版也只做只读。
- loadfs() 至少覆盖：
  1. 典型 AVDP 探测。
  2. 主路径失败时的必要后备锚点检查。
  3. 基础卷描述符与根目录建立。
- 新增 file_system_type fs_udf。
- 接入 probe 顺序时，确保最终顺序是：
  1. udf
  2. iso9660
  3. fat
- 这里说的是“最终顺序”，不是书写顺序；必须考虑 Register() 的头插行为。

当前细化顺序
1. 先做 UDF 存在性识别，不急着抢占现有 `iso9660` 路径。
2. 先把 UDF 的 root 句柄与只读框架补出来。
3. 等目录遍历足够稳定后，再把 `udf` 放到最终 probe 优先级最前。

完成标准
- 编译通过。
- 纯 UDF 镜像可识别并展开根目录。
- UDF Bridge 光盘优先识别为 udf，而不是误落到 iso9660。

验证方法
- 手动验证：
  1. 准备纯 UDF 镜像。
  2. 准备 UDF Bridge 镜像。
  3. 验证两者的 probe 结果和树展开结果。
- 回归验证：
  1. 纯 ISO9660/Joliet/RR 镜像行为不退化。

阶段 6：清理 ATAPI 设备信息与旧“猜 FAT”残留

目标
- 在新光盘文件系统路径可用后，清理与之冲突的历史行为。

修改范围
- D:\her\unisym\lib\cpp\Device\Storage\xCD.cpp
- D:\her\mecocoa\devdriv\storage\patadisk.cpp

实施内容
- 将 xCD_ATAPI::getSlice() 中旧的“猜 FAT/回落 FAT16”逻辑清理为中性 `sys_id`，让光盘识别完全依赖文件系统 probe 链。
- 确保 ATAPI whole-disk probe 不再依赖任何“伪 FAT 类型”才能进入识别。
- 收敛 ISO9660/UDF 的探测日志，避免非目标格式时逐扇区刷屏，同时保留足够的问题定位信息。
- 同步更新阶段说明，明确当前 `UDF` 完成口径仅覆盖单分区、`2048-byte logical block` 主路径。
- 清理阶段 0 可能留下的临时日志或兼容分支，但不要破坏已经验证通过的挂载入口。

完成标准
- 编译通过。
- 光驱识别结果完全由实际文件系统 probe 决定，不再依赖 xCD_ATAPI 的 FAT 猜测。
- 现有硬盘/分区路径不受影响。

验证方法
- 手动验证：
  1. ISO9660、Joliet、RR、UDF 各类镜像分别测试。
  2. 验证 probe 结果与实际格式一致。
- 回归验证：
  1. 普通 ATA 硬盘分区挂载不受影响。

建议的停顿点
- 阶段 0 完成后先停一次，确认控制流正确，再开始写新文件系统。
- 阶段 2 完成后先停一次。此时已经有“可用的 ISO9660 只读主路径”，可以先作为基线。
- 阶段 4 完成后先停一次。此时 ISO9660 家族基本成型，再决定是否继续把 UDF 纳入同一轮工程。

不建议的做法
- 不建议一上来就同时写 ISO9660、Joliet、Rock Ridge、UDF。
- 不建议为了接入新建一个 mecocoa 侧的大而泛的光盘桩文件。
- 不建议继续沿用“ATAPI 也伪装成某种 FAT sys_id”的策略来驱动识别。
- 不建议在首版就尝试写支持。

最终验收口径
- 光驱设备能被正确识别并进入 whole-disk probe。
- 至少一种常见 ISO9660 镜像能稳定挂载、列目录、读文件。
- 长文件名/中文名和 Unix 风格名称扩展分别有独立阶段收敛。
- UDF 是否纳入本轮，以阶段 4 完成后的实际效果和调试成本再决定。

---

如果你要做热插拔，下一阶段至少要补三块：

介质状态检测
要能知道“无盘 -> 有盘”或“有盘 -> 换盘/无盘”。

挂载生命周期管理
插盘后自动 probe/mount，拔盘后自动 unmount。

失效资源处理
已打开文件、缓存的 dentry、当前 cwd 指向光盘目录时怎么处理。



