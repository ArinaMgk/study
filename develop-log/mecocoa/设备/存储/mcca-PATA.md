# 20260629 为 PATA 等磁盘添加 GPT 支持：分阶段实施方案

目标：

让 PATA 等块设备同时支持 MBR 与 GPT，并把分区访问统一成同一种模型：

- `dev=0` 表示整盘
- `dev=1..N` 表示第 1..N 个分区

每个阶段完成时都要求：

1. 内核可正常编译
2. 当前阶段成果可单独验证
3. 已有 MBR 路径不被破坏

说明：

- 每阶段的编译与验证由用户执行
- 本方案只定义每阶段应达到的状态与建议验证内容


## 最终方案

最终采用下面这套结构：

1. `PartitionSlice` 继续保持轻量，只负责对外返回最常用的分区信息
2. `HD_Info` 改成统一分区表述，不再区分 `primary[]` 和 `logical[]`
3. MBR 与 GPT 都把结果写入同一个 `parts[]`
4. GPT 额外元数据单独侧存，不塞进 `PartitionSlice`
5. 运行期驱动与 prehost/loader 共用同一套分区探测和取 slice 逻辑

建议定义固定导出上限：

- `#define MAX_PARTITIONS 128`

这里的 `128` 表示：

- 内核对外缓存和导出的分区上限

并不表示：

- GPT 规范只允许 128 个分区项


## 阶段 1：重构 HD_Info，统一分区缓存格式

### 本阶段要做什么

修改 `D:\her\unisym\inc\cpp\trait\StorageTrait.hpp`。

把当前 `HD_Info`：

- `primary[NR_PRIM_PER_DRIVE]`
- `logical[NR_SUB_PER_DRIVE]`

改成统一结构，例如：

- `whole_disk`
- `scheme_kind`
- `part_count`
- `part_overflow`
- `parts[MAX_PARTITIONS]`

其中：

- `whole_disk` 保存整盘范围
- `scheme_kind` 标识当前盘是 `MBR` 还是 `GPT`
- `part_count` 表示当前有效分区数
- `part_overflow` 表示实际有效分区数是否超过导出上限
- `parts[]` 按平坦编号保存对外分区

`PartitionSlice` 本阶段不扩张，继续保持轻量。

如果 GPT 需要额外元数据，本阶段只预留侧表位置，例如：

- `gpt_meta[MAX_PARTITIONS]`

### 本阶段做完后应达到什么状态

1. `HD_Info` 已不再以主分区/逻辑分区数组作为核心表示
2. `PartitionSlice` 对外接口保持轻量
3. 后续 MBR/GPT 都可以往同一个 `parts[]` 里填
4. `MAX_PARTITIONS` 被明确定义为“导出上限”而非 GPT 项数假设

### 本阶段如何验证

由用户执行：

1. 编译 `unisym` 与 `mecocoa`
2. 确认所有直接依赖 `HD_Info` 布局的地方都已同步修正到可编译
3. 暂时不要求 GPT 行为，只要求结构改动收口


## 阶段 2：把 MBR 解析改写到统一 parts[] 中

### 本阶段要做什么

修改 `D:\her\unisym\lib\cpp\Device\Storage.cpp`。

保留现有 MBR 解析逻辑的语义，但把输出目标从：

- `primary[]`
- `logical[]`

改成：

- `whole_disk`
- `part_count`
- `parts[]`

要求：

1. `dev=0` 的信息由 `whole_disk` 表达
2. MBR 的主分区和逻辑分区最终都按“第几个分区”写入 `parts[]`
3. 对外不再保留主/逻辑分区区别
4. 若有效分区数超过 `MAX_PARTITIONS`，只导出前 `MAX_PARTITIONS` 个，并设置 `part_overflow`

这一阶段先不接入 GPT，只把 MBR 输出格式统一掉。

### 本阶段做完后应达到什么状态

1. MBR 盘已经可以被解析成平坦的 `parts[]`
2. 即使盘内有扩展分区链，对外也只看到连续编号分区
3. 后续 GPT 可直接复用同样的输出格式

### 本阶段如何验证

由用户执行：

1. 编译内核
2. 用已有 MBR 镜像启动
3. 确认 MBR 分区都能被正确写入 `parts[]`
4. 确认整盘信息仍正确


## 阶段 3：新增统一取分区接口，切换 dev 新语义

### 本阶段要做什么

在 `D:\her\unisym\lib\cpp\Device\Storage.cpp` 或合适头源文件中增加统一接口，例如：

- `PartitionSlice GetPartitionSlice(const HD_Info& hdi, unsigned dev);`

语义固定为：

- `dev=0` 返回整盘
- `dev=1..part_count` 返回第 N 个分区
- 超范围返回空 slice

然后开始替换直接读 `HD_Info` 内部布局的调用点，让它们统一走这个接口。

### 本阶段优先改哪些地方

先改运行期最核心路径：

- `D:\her\mecocoa\devdriv\storage\patadisk.cpp`

重点位置：

- `_LOCAL_GetPartitionSlice()`
- `GetPartitionSlice()`
- `Harddisk_PATA::getSlice()`

### 本阶段做完后应达到什么状态

1. 运行期已经按新 `dev` 语义取分区
2. 调用方不再自己解释 `primary/logical`
3. `getSlice(dev)` 已经真正变成“第几个分区”

### 本阶段如何验证

由用户执行：

1. 编译内核
2. 用 MBR 镜像启动
3. 验证：
   - `dev=0` 返回整盘
   - `dev=1..N` 返回正确分区
   - 超范围分区返回空


## 阶段 4：改 patadisk 的探测与挂载扫描

### 本阶段要做什么

继续修改 `D:\her\mecocoa\devdriv\storage\patadisk.cpp`。

把以下逻辑全部切到新模型：

1. `hd_open()` 只负责：
   - 读取 identify 信息
   - 填整盘容量
   - 调用统一分区探测函数

2. `print_hdinfo()` 改为遍历：
   - `whole_disk`
   - `part_count`
   - `parts[]`

3. 分区扫描挂载改为：
   - 遍历 `dev=1..part_count`
   - 不再使用旧的主分区/逻辑分区编号窗口

### 本阶段做完后应达到什么状态

1. `patadisk` 完全切到平坦分区模型
2. 当前运行期已不再依赖旧 MBR 设备号规则
3. MBR 分区仍能被扫描、探测、挂载

### 本阶段如何验证

由用户执行：

1. 编译内核
2. 启动 MBR 镜像
3. 验证：
   - 磁盘识别正常
   - 分区输出正常
   - FAT 分区仍可被探测并挂载


## 阶段 5：统一 prehost / loader 路径

### 本阶段要做什么

修改：

- `D:\her\mecocoa\prehost\atx-x86-flap32\atx-x86-flap32.loader.cpp`

要完成两件事：

1. prehost 使用和运行期一致的 `HD_Info` 结构
2. prehost 使用和运行期一致的 `GetPartitionSlice(dev)` 语义

也就是说，loader 不再自己展开：

- `primary[]`
- `logical[]`

而是直接走统一接口。

### 本阶段做完后应达到什么状态

1. 运行期和 prehost 的分区编号语义一致
2. 后续 GPT 支持接入后，早期路径不需要再单独重做一次

### 本阶段如何验证

由用户执行：

1. 编译涉及 loader 的目标
2. 用现有 MBR 启动链验证
3. 确认 loader 能按新 `dev` 语义访问分区


## 阶段 6：加入 GPT 探测与 GPT 分区解析

### 本阶段要做什么

修改 `D:\her\unisym\lib\cpp\Device\Storage.cpp`。

在统一分区探测函数中加入 GPT 路径：

1. 读取 LBA0，识别 protective MBR
2. 读取 LBA1，解析并校验 primary GPT header
3. 读取并校验完整 GPT entry array
4. 把 GPT 分区写入：
   - `part_count`
   - `parts[]`
   - `gpt_meta[]`

如果 primary GPT 无效，则继续：

5. 读取磁盘最后一个 LBA 的 backup GPT header
6. 若 backup GPT 有效，则按 backup GPT 导出分区，并将 `scheme_kind` 标记为 backup GPT 路径

### 这一阶段要保存哪些 GPT 信息

对外 `PartitionSlice` 仍只保存：

- `address`
- `length`
- `sys_id`

GPT 专有信息放入侧表，例如：

- type GUID
- unique GUID
- attributes

### 这一阶段 GPT 必须做的最小合法性校验

#### 1. 读取 LBA0

- 检查 MBR 末尾 `0x55AA`
- 检查是否存在 `type=0xEE` 的 protective MBR entry

#### 2. 读取 LBA1 primary GPT header

至少检查：

- `signature == "EFI PART"`
- `header_size` 合法，且不超过一个 logical block
- header CRC32 正确
- `MyLBA == 1`
- `AlternateLBA == last_lba`
- `FirstUsableLBA <= LastUsableLBA`
- `PartitionEntryLBA` 与 entry array 覆盖范围不越界

#### 3. 读取完整 GPT entry array

按 header 中字段处理：

- `NumberOfPartitionEntries`
- `SizeOfPartitionEntry`
- `PartitionEntryArrayCRC32`

要求：

1. 按 `NumberOfPartitionEntries * SizeOfPartitionEntry` 计算完整 entry array 长度
2. 按完整 entry array 做 CRC32 校验
3. 不因为 `parts[MAX_PARTITIONS]` 只缓存 128 项，就只读取前 128 项或只对前 128 项做 CRC

#### 4. 逐项检查有效 GPT entry

对每个非空 entry，至少检查：

- `start <= end`
- `start/end` 落在 `FirstUsableLBA..LastUsableLBA`
- 不与已导出分区重叠

重叠检查至少应在 debug 版保留。

### GPT 分区导出规则

1. 先按 header 指定的完整 entry array 读取并校验
2. 再遍历全部 entry，筛出有效分区
3. 只把前 `MAX_PARTITIONS` 个有效分区导出到：
   - `parts[]`
   - `gpt_meta[]`
4. 若有效分区超过 `MAX_PARTITIONS`，设置 `part_overflow`

这样：

- `parts[128]` 只是内核导出缓存上限
- 不会误把 GPT 规范理解成固定 128 项
- 不会因只读前 128 项而对非默认 GPT 做出错误 CRC 结论

### 本阶段做完后应达到什么状态

1. GPT 盘可被识别
2. GPT 分区可被导出为平坦 `dev=1..N`
3. MBR 与 GPT 都能通过统一接口取 slice
4. primary GPT 校验失败时，可尝试只读使用 backup GPT

### 本阶段如何验证

由用户执行：

1. 编译内核
2. 使用 GPT 镜像
3. 验证：
   - GPT Header 被识别
   - header CRC 正确时可正常通过
   - entry array CRC 正确时可正常通过
   - 分区数正确
   - 超过 `MAX_PARTITIONS` 时会设置 `part_overflow`
   - `dev=1..N` 可返回正确 GPT 分区范围
   - primary GPT 失效而 backup GPT 有效时，可进入 backup 路径


## 阶段 7：打通 GPT 到现有 FAT 挂载链

### 本阶段要做什么

修改：

- GPT 解析阶段的 `sys_id` 兼容映射
- `D:\her\mecocoa\mecocoa\filesys.cpp` 相关 probe 行为

先实现最小兼容桥：

1. 为常见 GPT 分区类型填写兼容 `sys_id`
2. 让现有 FAT probe 能继续识别 GPT 上的 FAT/ESP 分区

第一版先覆盖最常用场景即可：

- EFI System Partition
- 常见 FAT 数据分区

### 本阶段做完后应达到什么状态

1. GPT 上的 FAT/ESP 分区可以进入当前挂载探测流程
2. 至少一个 GPT 分区能实际 mount 成功

### 本阶段如何验证

由用户执行：

1. 编译内核
2. 用带 FAT/ESP 的 GPT 镜像启动或挂载测试
3. 验证：
   - probe 日志出现
   - 至少一个 GPT 分区可成功挂载


## 阶段 8：清理旧接口与残留 MBR-only 代码

### 本阶段要做什么

在 MBR 与 GPT 都跑通后，回头清理残留旧逻辑：

1. 删除不再需要的 `primary[]` / `logical[]` 相关旧接口
2. 删除旧编号窗口相关分支
3. 清理只为旧 MBR 模型服务的打印和扫描逻辑
4. 统一注释与使用约定

### 本阶段做完后应达到什么状态

1. 分区模型只剩一套
2. `dev=0 / dev=1..N` 成为唯一有效语义
3. MBR 与 GPT 共用同一条运行路径

### 本阶段如何验证

由用户执行：

1. 编译内核
2. 分别用 MBR / GPT 镜像验证
3. 确认双路径都正常，无旧接口残留依赖


## 实际开工顺序

按下面顺序推进：

1. 先改 `StorageTrait.hpp`，重构 `HD_Info`
2. 再改 `Storage.cpp`，让 MBR 输出进入统一 `parts[]`
3. 再加 `GetPartitionSlice(dev)` 新语义
4. 再改 `patadisk.cpp`
5. 再改 prehost/loader
6. 再接 GPT parser
7. 再补 GPT -> FAT 兼容桥
8. 最后清理旧接口


## 阶段收口要求

每一阶段结束都必须完成：

1. 由用户完成编译
2. 由用户启动当前可验证环境
3. 由用户验证本阶段目标
4. 由用户确认前一阶段成果未回退

如果某阶段改动过大，应拆小，不要把多个阶段混成一次提交。


## 当前实现状态

截至当前代码状态：

1. 阶段 1 至阶段 7 已完成，并已由用户进行编译和运行验证
2. 阶段 8 的主体也已完成

当前已经成立的结果：

1. `HD_Info` 已只保留统一分区模型：
   - `whole_disk`
   - `scheme_kind`
   - `part_count`
   - `part_overflow`
   - `parts[]`
   - `gpt_meta[]`
2. `PartitionSlice` 仍保持轻量
3. MBR 与 GPT 都导出到同一个 `parts[]`
4. 运行期 `Harddisk_PATA::getSlice(dev)` 已按：
   - `dev=0` 整盘
   - `dev=1..N` 第 N 个分区
5. `patadisk` 的挂载扫描已经按 `part_count` 连续遍历
6. GPT 解析已独立放在：
   - `D:\her\unisym\lib\cpp\Device\Storage\Storage-GPT.cpp`
7. GPT 上的 ESP/FAT 分区已能进入挂载流程并成功 mount
8. loader 保持 MBR-only，通过空的 `LoadGPTPartitions(...)` 避免链接 GPT 重实现

当前还保留、但已不再属于主路径的内容：

1. `FiledevMsg::GETPS` 枚举值仍保留，用作历史兼容占位
2. `fileman.hpp` 中旧的 `DRV_OF_DEV` / `MINOR_hd*a` 宏仍在，主要是为了不一次性牵动过多旧接口

因此，本方案的主目标已经达到：

- PATA 运行期已支持 GPT
- GPT 分区已能统一导出并接入现有挂载链
- MBR 与 GPT 已共用统一分区访问模型

# 20260716

先说结论：你现在这套 `IDE(PATADISK)` 代码离 UDMA 还差的是“整条 DMA 通路”，不只是换一个 ATA 命令。当前 [`D:\her\unisym\lib\cpp\Device\Storage\Harddisk.cpp`](D:/her/unisym/lib/cpp/Device/Storage/Harddisk.cpp:72) 里的 `Harddisk_PATA::Read/Write()` 还是直接对 `REG_DATA` 做 `IN_wn/OUT_wn`，而 [`D:\her\unisym\inc\c\storage\harddisk.h`](D:/her/unisym/inc/c/storage/harddisk.h:58) 的 `Harddisk_PATA` 也还没有任何 BMIDE/PRD/传输模式状态。`patadisk.cpp` 这边只是把它接进现有存储栈和 IRQ 服务里了，[`D:\her\mecocoa\devdriv\storage\patadisk.cpp`](D:/her/mecocoa/devdriv/storage/patadisk.cpp:51)。

我觉得可以考虑 4 种方案，按工程风险从低到高排：

1. `最小可落地方案：先做“PCI IDE Bus-Master DMA”，不急着自己切 UDMA 模式`
这版目标是先把 PIO 数据搬运改成 BMIDE DMA，命令改成 `READ DMA/WRITE DMA`，加 PRD 表、BMIDE command/status 寄存器和 IRQ 完成路径。
优点是改动集中，能明显减轻 CPU 搬运；而且很多虚拟机/BIOS 可能已经把盘设在某个 DMA/UDMA 模式上了，你主机侧只要能跑 DMA 就已经有价值。
缺点是“名义上支持 DMA”，但未必是你亲手完成“UDMA 模式协商”。
这方案最适合先做第一阶段 baseline。

2. `通用 IDE UDMA 方案：BMIDE DMA + IDENTIFY 解析 + SET FEATURES 传输模式协商`
这才是更完整的“支持 UDMA”。
大致要补：
`IDENTIFY DEVICE` 里解析支持的 `MWDMA/UDMA` 能力与当前活动模式；
启动时发 `SET FEATURES / Transfer Mode`，选择 `UDMA2/4/...`；
检查 80 芯线缆条件、主从共享通道约束、失败回退到较低 DMA 或 PIO。
优点是语义完整，真正能说“驱动支持 UDMA 模式选择”。
缺点是复杂度明显上升，错误面也大，尤其是不同 IDE 控制器对 timing/programming interface 的要求可能不完全一样。

3. `架构重整方案：把 PATA 拆成 channel/controller + device 两层，再加 DMA 后端`
也就是别继续把所有逻辑都塞在 `Harddisk_PATA` 里，而是引入更清晰的：
`IDE channel/controller`：primary/secondary、command block、control block、BMIDE base、IRQ、PRD workspace
`ATA/ATAPI device`：master/slave、IDENTIFY、读写命令、模式能力
然后做两个数据后端：
`PIO backend`
`DMA backend`
优点是后面 ATAPI DMA、多扇区 DMA、错误恢复、主从仲裁都会更顺。
缺点是第一步改动范围最大，不适合“先赶紧跑起来”。

4. `现实取舍方案：IDE 只补到 DMA baseline，追求高性能直接转 AHCI`
这不是回避问题，而是工程取舍。
你仓库里其实已经有一套 AHCI DMA 成熟路径了，比如 [`D:\her\unisym\inc\c\storage\AHCI.h`](D:/her/unisym/inc/c/storage/AHCI.h:192)、[`D:\her\unisym\lib\cpp\Device\Storage\Harddisk-SATA.cpp`](D:/her/unisym/lib/cpp/Device/Storage/Harddisk-SATA.cpp:173)、[`D:\her\mecocoa\devdriv\storage\ahcidisk.cpp`](D:/her/mecocoa/devdriv/storage/ahcidisk.cpp:28)。
如果你的真实目标是“让虚拟机/新机器上的磁盘高速稳定”，那 AHCI 往往比把 legacy IDE UDMA 做到很深更值。
缺点是它不能解决“我就是要 IDE/PATADISK 支持 UDMA”的这个目标本身。

如果是我来推进，我会推荐这个顺序：

1. 先做 `方案 1`
先把 `PATADISK` 跑通 BMIDE DMA，保留现有 PIO 作为 fallback。
这样你很快就能验证：
PCI IDE 节点是否暴露了 BMIDE I/O BAR
PRD/IRQ/完成路径是否稳定
现有 `Task_Hdd_Serv -> StorageTrait -> DiscPartition -> Filesys::Mount` 链路是否无需大改

2. 稳定后再上 `方案 2`
补 IDENTIFY 能力解析和 `SET FEATURES` 协商，把“DMA 能跑”提升成“UDMA 模式可控”。

3. 只有当你准备长期维护 IDE 栈时，才考虑 `方案 3`
不然容易一开始就把面铺太大。

---

20260717

所以如果你问“现在是不是 UDMA”：  是，按设备当前活动模式看，就是 UDMA5。

如果你问“代码有没有自己完整实现 UDMA 模式协商”：  还没有，现在实现的是 BMIDE DMA 数据通路。

还没专门做的主要是：
显式 SET FEATURES 模式协商
更系统的多盘位实机验证
更长期的压力/异常恢复验证
清理一部分探测型日志

