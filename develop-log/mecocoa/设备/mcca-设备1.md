
# 20260622

```
设备与总线探测、设备树管理方案

一、目标

建立统一的设备与总线探测框架，用于：

1. 接收平台发现层提供的硬件入口信息。
2. 调度各类总线扫描。
3. 维护统一的设备树。
4. 为驱动匹配、资源管理、设备初始化提供稳定骨架。


二、分层

1. 平台发现层

职责：
- 解析 RSDP、RSDT、XSDT、MADT、MCFG 等表。
- 提供平台级资源信息。

输出示例：
- LAPIC 列表
- PCIe ECAM 基址
- PCI bus range

2. 总线层

职责：
- 提供总线访问能力。
- 提供总线枚举能力。

PCI 层负责：
- CF8/CFC 配置空间访问
- ECAM/MMCONFIG 配置空间访问
- bus/device/function 扫描
- vendor/device/class/header 读取

3. 设备管理层

职责：
- 调度总线扫描。
- 建立和维护设备树。
- 组织驱动匹配与初始化顺序。
- 维护系统设备拓扑的高层视图。


三、设备树骨架

设备树节点采用以下形式：

	struct DeviceNode {
		Nnode link;
		DevExt fields;
	};

约定：

1. `link` 用作树骨架。
2. `fields` 用作设备语义区。
3. `Nnode` / `Nchain` 仅承担树链接与遍历职责。
4. 设备树不引入 parser 语义。


四、节点组织

树结构采用：

- `parent`
- `first child`
- `next sibling`

节点挂接关系：

1. 总线节点可挂设备节点。
2. 桥设备节点可挂子总线或子设备。
3. 控制器节点可挂其下游设备。
4. USB 设备与 USB interface 保留子节点扩展能力。


五、DevExt 设计

`DevExt` 承载设备身份、枚举信息、平台关联信息、驱动状态。

建议字段分组如下：

1. 节点身份
- node_type
- bus_type
- dev_class
- flags

2. 枚举信息
- vendor_id
- device_id
- revision
- class_base
- class_sub
- class_if

3. 地址身份
- pci_segment
- pci_bus
- pci_device
- pci_function

4. 平台关联
- acpi_handle

5. 驱动状态
- driver_name
- binding_state
- driver_data


六、资源模型

设备资源统一建模，不按单值字段散列保存。

资源结构采用统一形式：

	struct DeviceResource {
		uint16 type;
		uint16 flags;
		uint32 index;
		uint64 start;
		uint64 length;
		uint64 extra;
	};

资源容器单独约定。

可选形式：

1. 固定小数组 + count
2. Vector<DeviceResource*>
3. 资源子节点

第一阶段建议采用固定小数组或 Vector，并在 `DevExt` 中明确资源数量或资源容器边界。

资源类型覆盖：

1. MMIO BAR
2. I/O port BAR
3. IRQ line
4. MSI
5. MSI-X
6. DMA window
7. PCI bridge bus range
8. 其他可扩展资源


七、资源与状态的归属

1. 资源

以下内容归入 `DeviceResource`：
- BAR
- IRQ
- MSI
- MSI-X
- I/O port resource
- DMA window
- bus range

2. 身份与状态

以下内容归入 `DevExt`：
- ACPI handle
- vendor/device/class 信息
- PCI segment/bus/device/function
- driver binding
- binding state

3. 接口与下游对象

以下内容保留为节点扩展关系：
- USB device
- USB interface
- controller children


八、需要覆盖的设备情况

设备树方案必须直接支持以下情况：

1. 一个设备多个 BAR
2. 一个设备多个 IRQ
3. MSI / MSI-X
4. I/O port resource
5. DMA window
6. ACPI handle
7. PCI bridge bus range
8. USB interface
9. driver binding


九、Nchain 使用约定

`Nchain` 用于设备树骨架管理。

允许使用的能力：
- 节点挂接
- 子节点挂接
- 遍历
- 删除
- 释放

设备树代码采用设备管理专用约定，不引入 parser 风格操作习惯。

拓扑约定：

1. 树关系负责父子拓扑。
2. `DevExt` 只保存本节点自己的地址身份。
3. `DeviceResource` 负责窗口、范围与资源描述。
4. 不在每个节点中重复保存 parent bus、parent device 一类拓扑字段。
5. 如需缓存父总线号或桥信息，应视为缓存字段，而不是主身份字段。


十、设备树范围

第一阶段范围：

1. 建立系统根节点。
2. 建立 PCI 总线节点。
3. 挂接 PCI 设备节点。
4. 记录 PCI 资源。
5. 记录 bridge bus range。
6. 为 xHCI、USB 设备、USB interface 预留子节点路径。

后续扩展范围：

1. 控制器子树
2. USB 设备与 interface 子树
3. 驱动匹配状态
4. 资源仲裁
5. 初始化顺序管理


十一、推进顺序

1. 定义 `DeviceNode`
2. 定义 `DevExt`
3. 定义 `DeviceResource`
4. 建立设备树根节点
5. 接入 PCI 总线扫描结果
6. 生成 PCI 设备节点
7. 填充资源信息
8. 接入 bridge bus range
9. 接入控制器节点
10. 接入驱动绑定状态


十三、PCI 节点命名

PCI 节点命名采用完整地址身份。

建议形式：

- `pci@0000:00:1f.0`

不采用仅含 bus/device/function 的缩写形式。

命名字段对应：

- `pci_segment`
- `pci_bus`
- `pci_device`
- `pci_function`

十四、结论

正式方案如下：

1. 使用 `Nnode` / `Nchain` 作为设备树骨架。
2. 节点采用 `DeviceNode { Nnode link; DevExt fields; }` 形式。
3. `DevExt` 承载身份信息与状态信息，并保存完整 PCI 地址身份。
4. 资源采用统一的 `DeviceResource` 模型。
5. 树关系负责拓扑，资源模型负责窗口与范围，不重复保存父拓扑字段。
6. 设备管理层负责总线调度、设备树组织、驱动匹配与初始化顺序管理。
7. 平台发现层与总线层保持各自独立职责，不并入设备树核心结构。

```

这是一个非常清晰、务实且具有高度可扩展性的操作系统设备树与总线管理方案。从宏观架构到具体的数据结构切分，整体思路非常接近现代主流操作系统（如 Linux 的 `kobject`/`device` 模型或 Windows 的对象管理器框架），但又针对早期开发阶段做了合理的精简。

将“拓扑骨架”（`Nnode`）与“设备语义”（`DevExt`）分离，并且拒绝在内核运行时引入复杂的 Parser 语义，是非常明智的工程决策，这能极大降低内存开销和解析延迟。

以下是对该方案的架构验证、潜在挑战以及代码实现细节的一些技术性反馈：

## 架构亮点验证

- **职责边界清晰：** 将平台发现层（ACPI/MADT解析）、总线层（PCIe ECAM扫描）与设备管理层完全解耦。这意味着未来如果移植到 ARM 架构（使用 FDT/DeviceTree Blob 而不是 ACPI），你只需要替换平台发现层，设备管理层可以几乎无缝复用。
    
- **左子右兄树模型（Left-Child Right-Sibling）：** 配合 `Nnode` 中“长子节点的 `left` 指向父节点”的巧妙设计，既实现了任意多叉树（适合挂载多个设备的 USB Hub 或 PCIe Switch），又省去了一个专门的 `parent` 指针，对内核级内存管理非常友好。
    
- **标准的命名规范：** 采用 `pci@0000:00:1f.0` 这种类似 OpenFirmware 的命名方式，不仅方便日志调试，也为后续可能引入的 sysfs/procfs 虚拟文件系统节点提供了直接的路径映射。
    

## 潜在挑战与演进建议

### 1. 资源容器的动态性扩展

你在第一阶段建议对 `DeviceResource` 采用固定小数组或 Vector。对于大部分 PCI 设备（通常最多 6 个 BAR）是足够的。

- **边界情况：** 某些多功能设备、复杂的桥接设备，或者动态开启 SR-IOV（单根 I/O 虚拟化）的网卡，其资源数量可能会动态膨胀。
    
- **建议：** 如果内核堆分配器（Heap Allocator）已经稳定，Vector 是首选；但如果不希望在资源遍历时处理内存重新分配，未来可以将 `DeviceResource` 设计为侵入式链表（Intrusive List），这样挂接任意数量的资源都不会有容量上限，且不需要移动内存。
    

### 2. 驱动对象解耦（Driver Binding）

在 `DevExt` 中直接保存 `driver_name`、`binding_state` 和 `driver_data` 在初期非常直接有效。

- **演进方向：** 随着系统复杂化，多个相同型号的网卡会复用同一个驱动程序。未来可以引入独立的 `DriverObject` 结构。`DevExt` 中只需保留一个指向 `DriverObject` 的指针。这样可以更方便地进行驱动级别的引用计数管理（Reference Counting），并在驱动卸载时安全地解除所有关联设备的绑定。
    

### 3. 热插拔与并发控制（Hotplug & Concurrency）

目前方案主要针对冷启动枚举（Cold Boot Enumeration），比如静态的 PCI 扫描。

- **并发问题：** 当推进到第 10 步（USB 设备接入）或支持 Thunderbolt/PCIe 热插拔时，设备树的结构可能会在系统运行时突然改变。
    
- **机制预留：** 建议在 `DeviceNode` 或整个设备树管理器中预留读写锁（RW-Lock）或 RCU（Read-Copy-Update）机制的占位符。任何遍历 `Nchain` 的操作都必须在锁定状态下进行，防止在遍历链表时设备节点被物理移除并释放内存。
    

## 代码层面的观察 (Nnode 设计)

你提供的 `Nnode` C++ 实现非常紧凑。使用 `union` 来复用字段（例如 `union { Nnode* left, * pare; };`）是极限压榨内存的好方法。

需要特别注意的是这种设计的**类型安全陷阱**。因为你复用了字段，调用者如果错误地对长子节点调用了期望获取 sibling 的操作，就会意外拿到 parent 指针并导致逻辑死循环或崩溃。你已经在代码中提供了 `Nnode_getLeft` 和 `Nnode_getParent` 等内联/成员函数来进行安全访问，请务必在设备树操作代码中**严格限制**只通过这些封装好的接口来遍历树，绝对不要在设备管理层直接硬编码访问 `.left` 或 `.pare` 字段。

这个设备模型的骨架已经非常健壮，完全可以支撑起一个现代 OS 的驱动框架。

关于设备树运行时的动态变化，你目前计划如何在设备树中处理设备的动态生命周期管理（例如，当一个 USB 接口突然拔出时，如何安全地释放其对应的 `DeviceNode` 及其子节点，而不导致正在使用它的驱动崩溃）？

除了老旧的 8042/ISA 传统平台设备和现代的 PCI/PCIe 动态总线设备，在现代计算机系统（特别是 x86、ARM 等多架构体系）中，还存在以下几大类设备家族。

根据它们的**连接层级**和**硬件特性**，主要可以分为以下四类：

## 1. 处理器本地/内置设备 (CPU-Local Devices)

这类设备直接集成在 CPU 核心内部或紧邻核心，不需要通过外置的 PCI 总线进行寻址。它们通常通过特定的 CPU 寄存器（如 x86 的 MSR、ARM 的系统寄存器）或固定的极高优先级内存地址进行访问。

- **Local APIC / GIC (高级可编程中断控制器)：** 负责接收和分发中断。在 x86 中，Local APIC 拥有固定的物理地址（默认 `0xFEE00000`）；在 ARM 中则是 GIC。
    
- **IOMMU / SMMU (内存管理单元)：** 负责设备的 DMA 地址转换和内存保护（如 Intel VT-d、AMD-Vi）。
    
- **CPU 内置定时器：** 如 x86 的 APIC Timer、TSC（时间戳计数器），ARM 的 Generic Timer。
    

## 2. 现代嵌入式与片上系统总线设备 (SoC Platform Devices)

在 ARM、RISC-V 以及现代 Intel/AMD 的 SoC（片上系统）中，大量设备是直接硬连线在芯片内部总线上的（如 ARM 的 **AMBA/AXI/APB 总线**）。

它们和 ISA 设备一样**不可自枚举**，但它们是现代的高速设备。

- **常见设备：** 芯片内置的 UART（串口）、GPIO 控制器、I2C/SPI 总线控制器、看门狗定时器（Watchdog）、内置的电量管理单元（PMU）。
    
- **操作系统的发现机制：** 由于这类总线无法像 PCI 那样动态扫描，硬件厂商必须提供 **Device Tree（设备树/DTB）** 或 **ACPI 表（如 DSDT）**。操作系统在启动时解析这些文本或二进制表，才能知道这些设备被映射到了哪些 MMIO 地址、使用了哪些中断号。
    

## 3. 串行衍生总线设备 (Serial Bus Devices)

这类设备不直接暴露给 CPU 系统总线，而是“挂载”在某个总线控制器下面。CPU 如果想和它们通信，必须通过控制器的寄存器作为中介。

- **USB 设备 (Universal Serial Bus)：** 挂载在 **xHCI 控制器**（PCI 设备）之下。键盘、鼠标、U盘、摄像头。OS 驱动不直接操作这些设备，而是操作 xHCI 控制器的 Ring 缓冲区。
    
- **I2C / SPI / UART 从设备：** 挂载在对应的 SoC 控制器下。例如：主板上的触摸板、环境光传感器、EEPROM 固件芯片、硬件加密芯片（TPM 常常挂在 SPI 或 LPC 上）。
    

## 4. 传统/过渡期总线设备 (Legacy & Transition Buses)

这些总线在现代标准 PC 中已经基本退役，但在特定工业领域、老旧系统或某些特定架构中依然存在。

- **LPC (Low Pin Count) 设备：** ISA 总线的继任者。虽然物理引脚变少且高度串行化，但软件逻辑上完全兼容 ISA。现代主板上的 **Super I/O 芯片**（负责老式串口、并口、软盘控制器）就是通过 LPC 连接到南桥的。
    
- **PCMCIA / ExpressCard：** 早期笔记本电脑上使用的扩展卡接口，相当于外置的 ISA/PCI 槽，现已被 USB 和 Thunderbolt 彻底取代。
    

## 总结：操作系统眼中的设备全景

如果从**操作系统内核开发**的视角来看，世界上所有的设备最终只分为两大流派：

|**设备流派**|**发现机制**|**典型代表**|
|---|---|---|
|**可动态枚举设备 (Enumerable)**|硬件自带配置空间，支持运行时扫描、发现和动态分配资源。|PCI/PCIe 设备、USB 设备、SATA/NVMe 存储设备。|
|**不可动态枚举设备 (Non-enumerable)**|硬件资源在硬件设计时就已固定，内核必须通过“先验知识”才能访问。|8042/ISA 传统设备、Local APIC、SoC 内部片上外设（依赖设备树/ACPI）。|

# 20260623

```
mgc 应该如何表示表格比较好？列举几个方案？

方案二看起来不错 ，让我们继续讨论这个。除此之外，我还想到可以导入外部 csv 等格式（不过csv 不支持跨行跨列复杂格式）

你看看这样行不行，复不复杂：

Table(TableName, 3, 4);
TableName[0][0].text = "aaa";
TableName[1][0].text = "aaa";// row2 col1
Out(TableName);
```

---

```
下一步应该做什么？怎么根据现有的设备树判断是什么类型设备并分配驱动呢？
```

下一步就该做：**设备匹配与驱动绑定表**。

你现在已经有了三样关键输入：

- 拓扑
- 设备身份
- 设备资源

也就是每个 `DeviceNode` 里已经有：

- `bus_type`
- `vendor_id / device_id`
- `class_base / class_sub / class_if`
- `pci_segment / bus / device / function`
- `resources`
- `binding`

所以“判断设备类型并分配驱动”已经有基础了。

我建议按这个顺序推进。

**第一步：先定义匹配规则**
不要一开始搞复杂框架，先定最常用的三层匹配优先级：

1. `vendor_id + device_id`
2. `class_base + class_sub + class_if`
3. `class_base + class_sub`

比如：

- `1033:0194` 或 `0C.03.30` -> `xHCI`
- `8086:100E` 或 `02.00.00` -> `e1000`
- `01.06.01` -> `AHCI`
- `03.00.00` -> `VGA`

这样先能把“这是什么设备”稳定判断出来。

**第二步：定义驱动描述表**
在 `devsman.cpp` 或单独文件里先做一个静态表，例如概念上：

```cpp
struct DriverMatchEntry {
	DeviceBusType bus_type;
	uint16 vendor_id;
	uint16 device_id;
	uint8 class_base;
	uint8 class_sub;
	uint8 class_if;
	const char* driver_name;
	bool (*probe)(DeviceNode&);
};
```

然后放几条最小规则。

**第三步：遍历设备树做绑定**
新增一个像这样的流程：

- 从 `Devsman::Root()` 开始遍历
- 只处理 `DeviceNodeType::PciDevice`
- 依次按上面的优先级匹配
- 匹配成功后写入：
  - `node.fields.dev_class`
  - `node.fields.binding.driver_name`
  - `node.fields.binding.state`

先不一定马上初始化驱动，先把“绑定结果”写进树。

**第四步：把绑定和初始化分两阶段**
这是关键。

不要一匹配上就立刻做所有初始化。建议分成：

1. `BindDrivers()`
   只判断“谁应该管这个设备”

2. `StartDrivers()`
   对已经绑定的节点调用驱动初始化

这样后面设备树会更稳，也方便调试。

**怎么根据现有树判断类型**
就用你现在已有字段：

- 判断是不是桥：`class_base == 0x06 && class_sub == 0x04`
- 判断是不是 xHCI：`class_base == 0x0C && class_sub == 0x03 && class_if == 0x30`
- 判断是不是网卡：`class_base == 0x02 && class_sub == 0x00`
- 判断是不是 VGA：`class_base == 0x03 && class_sub == 0x00`
- 判断是不是 AHCI：`class_base == 0x01 && class_sub == 0x06 && class_if == 0x01`

更细时再加 `vendor/device`。

**我建议你现在的最具体下一步**
不是马上写“通用驱动系统”，而是先实现：

**`Devsman::BindPCIDrivers()`**

完成标准：

- 遍历所有 `PciDevice`
- 按 class/vendor/device 匹配
- 把匹配结果写进 `binding.driver_name`
- 串口/调试输出类似：
  - `Bind pci@0000:00:04.0 -> xhci`
  - `Bind pci@0000:00:03.0 -> e1000`
  - `Bind pci@0000:00:11.0 -> pci-bridge`

这一步做完，你的设备树就从“枚举树”变成“设备管理树”了。  
然后再做下一步：让 `xHCI` 不再手动查 class，而是直接找“已绑定 xhci 驱动的节点”。

# 20260626


| 方向                      | 做什么                                     | 作用                            | 直接收益                                 | 调试价值               | 实现成本 | 风险                                   |
| ----------------------- | --------------------------------------- | ----------------------------- | ------------------------------------ | ------------------ | ---- | ------------------------------------ |
| `usb-dev` 补字符串描述符       | 读取并挂出 `manufacturer / product / serial` | 让设备节点从 `vid/pid` 变成“人能直接认”的设备 | 设备树可读性明显提升，看到名字就知道是什么设备              | 很高，排查多 USB 设备时特别有用 | 中    | 低到中。要补 string descriptor 读取和编码处理     |
| `usb-if` 补 endpoint 级信息 | 把 interface 下的 endpoint 配置挂出来           | 让接口节点更完整，体现 USB 通信结构          | 更接近完整 USB 拓扑，后面做 class driver/传输分析更顺 | 中，主要对驱动开发和协议排查有用   | 中到偏高 | 中。要从 endpoint/config 数据里整理结构，输出也会变复杂 |

一句话总结：

- **字符串描述符** 更偏“设备识别”和“树好读”
- **endpoint 信息** 更偏“协议/驱动开发细节”

有，而且可以按“从现在这棵树继续自然长出来”的顺序看。

| 方向                   | 做什么                                                        | 作用                        | 收益                  |
| -------------------- | ---------------------------------------------------------- | ------------------------- | ------------------- |
| USB hub / 多级拓扑       | 区分 root hub、外接 hub、下游设备                                    | 让 USB 树从“一层设备”变成真实多级拓扑    | 后面接 U 盘、hub、多设备会更自然 |
| 设备断开/移除              | 热拔插时把 `usb-dev` / `usb-if` 从树里摘掉                           | 让设备树反映真实在线设备，而不是只增不减      | 对长期运行和热插拔很重要        |
| 设备状态细化               | `matched/probed/started` 之外补 `disconnected/error` 等        | 让树能表达运行态，不只是启动态           | 调试价值高               |
| `usb-if` 下挂 endpoint | 把 interrupt/bulk/control endpoint 也表示出来                    | 更接近完整 USB 协议结构            | 方便驱动开发和传输调试         |
| 字符串描述符               | `manufacturer/product/serial`                              | 让设备更容易识别                  | 可读性和排障体验最好          |
| USB class 继续扩        | 不只 HID，后面支持 mass storage / hub / CDC 等                     | 让 USB 树和驱动绑定继续扩展          | 真正走向通用 USB 子系统      |
| 更通用的 driver binding  | 让 USB 也像 PCI 一样有独立 match/probe/start 表                     | 从“专门处理 HID”升级成通用 USB 驱动框架 | 架构更完整               |
| 设备节点与驱动对象关联清晰化       | 规范 `driver_data` / 生命周期 / ownership                        | 避免后面越做越乱                  | 对可维护性很关键            |
| 热插拔事件通知              | 设备树变化时给 shell/GUI/服务发通知                                    | 让上层能响应设备出现/消失             | 用户态体验更完整            |
| 统一资源/属性模型            | 设备属性不只塞进 `class/vendor/device`，而有更通用 key/value 或 typed ext | 避免后面各种总线都挤同一套字段           | 对多总线扩展有帮助           |

如果只讲“下一步最自然”，我会把优先级排成：

1. 字符串描述符  
2. 设备断开/移除  
3. hub / 多级拓扑  
4. `usb-if` 下 endpoint  
5. USB class 扩展和通用 binding

也就是说，接下来最值得做的已经不只是“打印更好看”，而是开始让这棵 USB 树具备**动态生命周期**。

**部分实现**

- 平台发现层
    - RSDP / MADT / MCFG 已解析
    - ECAM 已支持
    - 但还不是一个很完整、统一的“平台发现服务接口”
- 总线层
    - PCI 很完整
    - USB 目前够用，但还是偏 xHCI/HID 路径，不是完整通用 USB core
- 驱动初始化顺序管理
    - 已有 Bind / Probe / Start
    - 但目前真正跑通的 starter 还主要是 xhci
- USB interface 支持
    - 节点已经有了
    - 但 endpoint、更多 class 还没继续展开
- 资源类型覆盖
    - 已有 BAR / IRQ / IOPORT / bridge bus range / USB-LOC
    - 但 MSI / MSI-X / DMA window 还没进设备树资源模型

**还没实现**

- 资源仲裁
- 通用初始化顺序管理策略
- 通用控制器子树框架
- 完整 USB 设备/接口/endpoint 子树
- 设备删除/热插拔回收
- MSI / MSI-X / DMA window 作为设备树资源节点
- 不重复保存父拓扑这条原则虽然总体遵守了，但 USB 这块后面还要继续审视有没有临时缓存字段越界

```
现在有没有必要将 devsman 设备管理 升华成一个单独的内核线程
```

**现在先不要把 devsman 做成单独内核线程。**  
更自然的下一步是先把它做成一个**被动的、事件驱动的管理模块**：

- 继续用函数接口接收“设备已完成配置/已移除”
- 先补设备移除、热插拔、节点回收
- 等事件真的多起来，再考虑收成专门线程

---

√  `usb-dev` 补字符串描述符
√  `usb-if` 补 endpoint 级信息
√  设备断开/重新连接

---

现在先**不用马上做完整稳定 id 系统**。  
更合适的是分两步：

1. **先补一个轻量 runtime id**
    - 单调递增
    - 只用于当前运行期日志和节点区分
2. **等真需要持久识别时，再补 stable_id**
    - 优先用 controller BDF + port path
    - 有 serial 时再增强

# 20260627


**已经有的**

- PCI 基本枚举
    - bus/device/function
    - vendor/device
    - class/subclass/interface
    - BAR
    - IRQ line/pin
    - bridge bus range
- 设备树骨架
    - pci-root / usb-root / platform-root ...
- USB 基本枚举
    - usb-bus
    - usb-root-hub
    - usb-port
    - usb-dev
    - usb-if
    - USB-EP
- xHCI + hub + 下游设备
    - 冷启动
    - 热插拔
    - 拔整个 hub / 重接

**总线侦测还缺的**

1. **PCI bridge 后续总线扫描再做扎实**
    - VMware 场景下的 PCI bridge 后续总线扫描不完整，导致 xHCI 控制器根本没进树。
2. **更多总线的真正发现链**
    - 现在 i2c-root / spi-root / virtio-root / serio-root 更像骨架
    - 还没形成和 PCI/USB 一样完整的“发现 -> 挂树 -> 绑定”
3. **ACPI/平台设备枚举再往前走**
    - 现在 platform 主要还是手工挂了一些已知设备
    - 还没把 ACPI namespace 里的平台设备系统化进树

**设备侦测还缺的**

1. **PCI 设备识别覆盖面继续扩**
    - 现在真正深入处理的还是：
        - xhci
        - pata
        - 少量 platform/serio
    - 还缺：
        - NVMe
        - AHCI
        - e1000/vmxnet3/virtio-net
        - 音频
        - 更多显示/输入控制器
2. **USB class 扩展**
    - 现在主要是：
        - hub
        - HID keyboard
        - HID mouse
    - 还缺：
        - mass storage
        - composite device
        - CDC
        - audio
        - camera
        - BT dongle
3. **设备消失/重现后的完整生命周期**
    - USB 这块已经不错了
    - 但别的总线还没形成一样完整的：
        - attach
        - detach
        - re-enumerate

**如果只问“接下来最值得做什么”**  
我会按这个顺序排：

1. **把 PCI bridge 后续总线扫描做扎实**
    - 因为这会直接影响“能不能看见设备”
    - 比继续扩某个具体驱动更基础
2. **扩 PCI 存储/网络设备识别**
    - 例如 AHCI/NVMe、网卡
    - 这样设备树的实用性会明显提升
3. **扩 USB class**

- 先从 mass storage 开始最有价值
- VMware 场景下的 PCI bridge 后续总线扫描不完整，导致 xHCI 控制器根本没进树。


| 项目                   | 现在状态   | 说明                                                                                                                        |
| -------------------- | ------ | ------------------------------------------------------------------------------------------------------------------------- |
| 设备状态细化               | `部分实现` | 现在主要有 `matched / probed / started / failed`。USB 热插拔能增删树，但还没有系统化的 `disconnected / removed / error / stale` 这类更细状态。         |
| `usb-if` 下挂 endpoint | `已实现`  | 现在 `usb-if@0` 下面已经有 `USB-EP[...] addr/type/mps/interval`。                                                                 |
| 字符串描述符               | `已实现`  | `usb-dev` 已经能显示 `manufacturer / product / serial`。                                                                        |
| USB class 继续扩        | `部分实现` | 现在真正做通的是 `hub`、`HID keyboard`、`HID mouse`。还没扩到 `mass storage / CDC / audio / composite` 等。                                |
| 更通用的 driver binding  | `部分实现` | `Devsman` 已经有统一的 `Bind / Probe / Start` 框架，也有 driver match/start hook；但 USB 这边还不是完整通用框架，还是偏 `xhci/hub/hid` 定向打通。          |
| 设备节点与驱动对象关联清晰化       | `部分实现` | 现在已经普遍用 `binding.driver_data` 关联节点与运行对象，USB 这轮也在按 `driver_data` 找节点；但生命周期和 ownership 还没完全统一抽象。                            |
| 热插拔事件通知              | `部分实现` | 现在内核内部已经有热插拔驱动链和日志，树也会更新；但还没有更正式的“统一事件通知接口”给上层/用户态。                                                                       |
| 统一资源/属性模型            | `部分实现` | 现在已经有统一 `DeviceResource`，并覆盖了 `BAR / IRQ / BUS-RANGE / USB-LOC / USB-EP`；但属性层还比较分散，很多身份字段仍直接放在 `DevExt` 里，不是完全统一的可扩展属性体系。 |


第一阶段设备开发告一段落。

# 20260716 设备树一览

```
vmware 简化输出
=== Device Tree ===
system-root <system-root>
  pci-root <pci-root bus=pci>
    pci-bus@0000:00 <pci-bus seg=0 bus=0>
      pci@0000:00:00.0 <pci-dev 06.00.00 vend=8086 dev=7190>
      pci@0000:00:01.0 <pci-dev 06.04.00 vend=8086 dev=7191 drv=pci-bridge state=matched>
      pci@0000:00:07.0 <pci-dev 06.01.00 vend=8086 dev=7110>
      pci@0000:00:07.1 <pci-dev 01.01.8A vend=8086 dev=7111 drv=pata state=matched>
        pata-disk@0:0 <storage-dev bus=pci drv=pata-disk state=started>
        pata-cd@1:0 <storage-dev bus=pci drv=pata-cdrom state=started>
      pci@0000:00:07.3 <pci-dev 06.80.00 vend=8086 dev=7113>
      pci@0000:00:07.7 <pci-dev 08.80.00 vend=15AD dev=0740>
      pci@0000:00:0f.0 <pci-dev 03.00.00 vend=15AD dev=0405 drv=video-vmware state=started>
      pci@0000:00:10.0 <pci-dev 01.00.00 vend=104B dev=1040>
      pci@0000:00:11.0 <pci-dev 06.04.01 vend=15AD dev=0790 drv=pci-bridge state=matched>
        pci-bus@0000:02 <pci-bus seg=0 bus=2>
          pci@0000:02:00.0 <pci-dev 02.00.00 vend=1022 dev=2000>
          pci@0000:02:01.0 <pci-dev 01.06.01 vend=15AD dev=07E0 drv=ahci state=started>
            ahci-disk@0 <storage-dev bus=pci drv=ahci-disk state=started>
          pci@0000:02:02.0 <pci-dev 0C.03.00 vend=15AD dev=0774>
          pci@0000:02:03.0 <pci-dev 0C.03.20 vend=15AD dev=0770>
      pci@0000:00:15.0 <pci-dev 06.04.00 vend=15AD dev=07A0 drv=pci-bridge state=matched>
        pci-bus@0000:03 <pci-bus seg=0 bus=3>
          pci@0000:03:00.0 <pci-dev 0C.03.30 vend=15AD dev=077A drv=xhci state=probed>
      pci@0000:00:15.1 <pci-dev 06.04.00 vend=15AD dev=07A0 drv=pci-bridge state=matched>
      pci@0000:00:15.2 <pci-dev 06.04.00 vend=15AD dev=07A0 drv=pci-bridge state=matched>
      pci@0000:00:15.3 <pci-dev 06.04.00 vend=15AD dev=07A0 drv=pci-bridge state=matched>
      pci@0000:00:15.4 <pci-dev 06.04.00 vend=15AD dev=07A0 drv=pci-bridge state=matched>
      pci@0000:00:15.5 <pci-dev 06.04.00 vend=15AD dev=07A0 drv=pci-bridge state=matched>
      pci@0000:00:15.6 <pci-dev 06.04.00 vend=15AD dev=07A0 drv=pci-bridge state=matched>
      pci@0000:00:15.7 <pci-dev 06.04.00 vend=15AD dev=07A0 drv=pci-bridge state=matched>
      pci@0000:00:16.0 <pci-dev 06.04.00 vend=15AD dev=07A0 drv=pci-bridge state=matched>
      pci@0000:00:16.1 <pci-dev 06.04.00 vend=15AD dev=07A0 drv=pci-bridge state=matched>
      pci@0000:00:16.2 <pci-dev 06.04.00 vend=15AD dev=07A0 drv=pci-bridge state=matched>
      pci@0000:00:16.3 <pci-dev 06.04.00 vend=15AD dev=07A0 drv=pci-bridge state=matched>
      pci@0000:00:16.4 <pci-dev 06.04.00 vend=15AD dev=07A0 drv=pci-bridge state=matched>
      pci@0000:00:16.5 <pci-dev 06.04.00 vend=15AD dev=07A0 drv=pci-bridge state=matched>
      pci@0000:00:16.6 <pci-dev 06.04.00 vend=15AD dev=07A0 drv=pci-bridge state=matched>
      pci@0000:00:16.7 <pci-dev 06.04.00 vend=15AD dev=07A0 drv=pci-bridge state=matched>
      pci@0000:00:17.0 <pci-dev 06.04.00 vend=15AD dev=07A0 drv=pci-bridge state=matched>
      pci@0000:00:17.1 <pci-dev 06.04.00 vend=15AD dev=07A0 drv=pci-bridge state=matched>
      pci@0000:00:17.2 <pci-dev 06.04.00 vend=15AD dev=07A0 drv=pci-bridge state=matched>
      pci@0000:00:17.3 <pci-dev 06.04.00 vend=15AD dev=07A0 drv=pci-bridge state=matched>
      pci@0000:00:17.4 <pci-dev 06.04.00 vend=15AD dev=07A0 drv=pci-bridge state=matched>
      pci@0000:00:17.5 <pci-dev 06.04.00 vend=15AD dev=07A0 drv=pci-bridge state=matched>
      pci@0000:00:17.6 <pci-dev 06.04.00 vend=15AD dev=07A0 drv=pci-bridge state=matched>
      pci@0000:00:17.7 <pci-dev 06.04.00 vend=15AD dev=07A0 drv=pci-bridge state=matched>
      pci@0000:00:18.0 <pci-dev 06.04.00 vend=15AD dev=07A0 drv=pci-bridge state=matched>
      pci@0000:00:18.1 <pci-dev 06.04.00 vend=15AD dev=07A0 drv=pci-bridge state=matched>
      pci@0000:00:18.2 <pci-dev 06.04.00 vend=15AD dev=07A0 drv=pci-bridge state=matched>
      pci@0000:00:18.3 <pci-dev 06.04.00 vend=15AD dev=07A0 drv=pci-bridge state=matched>
      pci@0000:00:18.4 <pci-dev 06.04.00 vend=15AD dev=07A0 drv=pci-bridge state=matched>
      pci@0000:00:18.5 <pci-dev 06.04.00 vend=15AD dev=07A0 drv=pci-bridge state=matched>
      pci@0000:00:18.6 <pci-dev 06.04.00 vend=15AD dev=07A0 drv=pci-bridge state=matched>
      pci@0000:00:18.7 <pci-dev 06.04.00 vend=15AD dev=07A0 drv=pci-bridge state=matched>
  usb-root <bus-root bus=usb>
  platform-root <bus-root bus=platform>
    video-classic <platform-dev bus=platform drv=classic-video-driver state=started>
    pic@8259-master <platform-dev bus=platform drv=pic-8259 state=started>
    pic@8259-slave <platform-dev bus=platform drv=pic-8259 state=started>
    lapic@0 <platform-dev bus=platform drv=lapic state=started>
    ioapic@0 <platform-dev bus=platform drv=ioapic state=started>
    fdc@0 <platform-dev bus=platform>
      floppy@0 <storage-dev bus=platform>
    uart@com1 <platform-dev bus=platform drv=uart-8250 state=matched>
    pit@8254 <platform-dev bus=platform drv=pit-8254 state=started>
    rtc@cmos <platform-dev bus=platform drv=rtc-cmos state=matched>
  i2c-root <bus-root bus=i2c>
  spi-root <bus-root bus=spi>
  serio-root <bus-root bus=serio>
    i8042 <serio-ctl bus=serio drv=i8042 state=matched>
      ps2kbd <serio-dev bus=serio drv=ps2-keyboard state=matched>
      ps2mouse <serio-dev bus=serio drv=ps2-mouse state=matched>
  virtio-root <bus-root bus=virtio>

```

| 设备路径               | 厂商ID (vend)       | 设备ID (dev) | 设备名称 / 功能                                 |
| ------------------ | ----------------- | ---------- | ----------------------------------------- |
| `pci@0000:00:00.0` | `8086` (Intel)    | `7190`     | **Intel 82443BX 主桥/控制器**                  |
| `pci@0000:00:07.0` | `8086` (Intel)    | `7110`     | **Intel 82371AB PIIX4 ISA 桥**             |
| `pci@0000:00:07.3` | `8086` (Intel)    | `7113`     | **Intel 82371AB/EB/MB PIIX4 ACPI** (电源管理) |
| `pci@0000:00:07.7` | `15AD` (VMware)   | `0740`     | **VMware 虚拟机通信接口 (VMCI)**                 |
| `pci@0000:00:10.0` | `104B` (BusLogic) | `1040`     | **BusLogic BT-958 SCSI 控制器**              |
| `pci@0000:02:00.0` | `1022` (AMD)      | `2000`     | **AMD Lance/PCI 网卡**                      |
| `pci@0000:02:02.0` | `15AD` (VMware)   | `0774`     | **VMware USB 1.1 控制器** (UHCI)             |
| `pci@0000:02:03.0` | `15AD` (VMware)   | `0770`     | **VMware USB 2.0 控制器** (EHCI)             |

QEMU 当前未识别设备：

| 设备路径               | 厂商ID (vend)    | 设备ID (dev) | 设备名称 / 功能                                 |
| ------------------ | -------------- | ---------- | ----------------------------------------- |
| `pci@0000:00:00.0` | `8086` (Intel) | `1237`     | **Intel 440FX 主桥/内存控制器**                  |
| `pci@0000:00:01.0` | `8086` (Intel) | `7000`     | **Intel 82371SB PIIX3 ISA 桥**             |
| `pci@0000:00:01.3` | `8086` (Intel) | `7113`     | **Intel 82371AB/EB/MB PIIX4 ACPI** (电源管理) |
| `pci@0000:00:03.0` | `8086` (Intel) | `100E`     | **Intel PRO/1000 千兆网卡**                   |

声卡通常作为PCI设备被添加到虚拟机上。QEMU支持模拟多种声卡，例如：
- **Creative SoundBlaster 16**
- **ENSONIQ AudioPCI ES1370**
- **Intel 82801AA AC'97 Audio**
- **Intel HD Audio Controller**
- **VirtIO Sound** (一种半虚拟化声卡)

- [ ] 设备树映射 /dev
- [ ] 不回退实地址模式切换分辨率

till +8=18:00

---


