
# 20260806

```
我现在想实现设备IO接口(虚拟设备)，初步方案是 C:\Users\phina\Downloads\vdev.md

请只读评估：
1、这个方案
2、实现这个方案需要的工程量大不大
```



```
你看看我的这个想法好不好，让一切皆文件。把设备树的层级结构等价映射在 /dev 中。例如
system-root <system-root>
  pci-root <pci-root bus=pci>
    pci-bus@0000:00 <pci-bus seg=0 bus=0>
...
          fdc@0 <platform-dev bus=isa>
            floppy@0 <storage-dev bus=isa>
...
中的floppy@0可以被映射为 /dev/pci/pci-bus@0000:00/fdc@0/floppy@0

---

那我们就用这样两套试图。
1、/dev 采用Linux那套风格
2、/.dev 使用设备树等价动态映射 
3、/mnt 中的每个挂载点，都应该能追溯到 /.dev 中的某个存储设备节点
这个想法怎么样

```

## 概述

Device Abstraction Layer 用于在内核中建立统一的设备模型，使不同类型的设备能够以相同方式被管理和访问。

设备抽象层位于硬件驱动之上。驱动负责处理具体硬件，例如寄存器访问、DMA、中断以及设备协议；设备抽象层负责维护设备对象、设备之间的关系，并向内核其他模块提供统一接口。

设备抽象层不区分设备是否真实存在于硬件中。PCI 设备、USB 设备、NVMe Namespace、Framebuffer 等均可以作为 Device 存在。

例如：

```
PCI Bus
 |
 +-- NVMe Controller
       |
       +-- Namespace 0
       +-- Namespace 1
```

其中 PCI Bus 是总线设备，NVMe Controller 是物理设备，Namespace 是由控制器产生的逻辑设备，它们都属于设备模型的一部分。

---

## Device 模型

设备抽象层的核心对象为 Device。

```cpp
struct Device
{
    dev_t id;

    int type;
    int subtype;

    char name[32];

    Device *parent;

    void *private_data;

    DeviceOps *ops;
};
```

Device 表示系统中的一个可管理对象。

其中：

* `id` 作为系统内部引用设备的标识。
* `type/subtype` 用于描述设备类别。
* `parent` 用于表示设备拓扑关系。
* `private_data` 保存驱动自己的设备状态。
* `ops` 描述该设备向系统提供的操作。

设备抽象层只管理这些通用信息，不理解 `private_data` 的具体内容。

例如 NVMe 驱动可以保存：

```cpp
NVME_Controller *
```

UART 驱动可以保存：

```cpp
UART_Device *
```

设备层只负责在调用操作时将其传递给驱动。

---

## Device Operations

设备通过操作接口向系统提供能力。

基础操作定义：

```cpp
struct DeviceOps
{
    int (*f_read)(
        Device *,
        void *,
        size_t,
        idx_t,
        int
    );

    int (*f_send)(
        Device *,
        void *,
        size_t,
        idx_t,
        int
    );

    int (*f_ctrl)(
        Device *,
        int,
        void *,
        int
    );
};
```

`f_read` 用于设备数据读取。

例如：

* Block Device 读取指定块。
* UART 获取接收数据。
* Input Device 获取输入事件。

`f_send` 用于向设备发送数据。

例如：

* Block Device 写入数据。
* UART 发送字符。
* Network Device 发送数据包。

`f_ctrl` 用于设备控制操作。

例如：

* 获取设备信息。
* 修改设备参数。
* 执行设备特殊命令。

设备抽象层不会定义所有设备共有的行为，而只提供最基础的访问模型。不同类型设备可以根据自身需求扩展操作接口。

---

## 设备注册

驱动初始化完成后，通过设备安装接口向系统注册设备。

```cpp
dev_t device_install(
    int type,
    int subtype,
    void *ptr,
    char *name,
    dev_t parent,
    void *f_ctrl,
    void *f_read,
    void *f_send
);
```

注册时需要提供：

* 设备类型。
* 父设备。
* 驱动私有数据。
* 设备操作函数。

例如 NVMe 初始化完成后：

```cpp
device_install(
    DEV_BLOCK,
    BLOCK_NVME,
    controller,
    "nvme0",
    pci_device,
    nvme_ctrl,
    nvme_read,
    nvme_write
);
```

设备注册完成后，其他内核模块不需要知道 NVMe 的实现细节，只需要通过设备号访问。

---

## 设备访问

设备抽象层提供统一访问入口：

```cpp
int device_read(
    dev_t dev,
    void *buf,
    size_t count,
    idx_t idx,
    int flags
);


int device_send(
    dev_t dev,
    void *buf,
    size_t count,
    idx_t idx,
    int flags
);


int device_ctrl(
    dev_t dev,
    int cmd,
    void *args,
    int flags
);
```

调用过程：

```
Kernel Subsystem

        |
        v

device_read()

        |
        v

DeviceOps::f_read()

        |
        v

Driver

        |
        v

Hardware
```

这样文件系统、网络栈、终端系统等模块只依赖设备抽象层，而不直接依赖具体驱动。

---

## 设备层级关系

设备之间通过 parent 建立关系。

例如：

```
System
 |
 PCI Root
 |
 PCI Bus
 |
 NVMe Controller
 |
 Namespace
```

这种关系用于：

* 表示硬件拓扑。
* 管理设备生命周期。
* 进行资源继承。

例如 Namespace 的父设备是 NVMe Controller，因此 Namespace 可以继承 Controller 提供的资源。

---

## 总线设备

总线设备也是 Device。

但是总线设备的主要作用不是数据传输，而是发现和管理子设备。

例如 PCI：

```
PCI Bus

    |
    +-- scan()

    |
    +-- PCI Device
```

因此总线设备需要额外能力：

```cpp
struct BusOps
{
    int (*scan)(Device *);

    int (*probe)(Device *);

    int (*remove)(Device *);
};
```

BusOps 是 DeviceOps 的扩展，而不是另一套完全独立的设备模型。

PCI、USB、I2C 等总线通过该接口负责发现设备，并将发现的设备注册到设备抽象层。

---

## 虚拟设备

设备抽象层允许驱动创建虚拟设备。

例如：

```
NVMe Controller

        |
        |
        v

NVMe Namespace

        |
        |
        v

Block Device
```

Namespace 并不是一个独立 PCI 硬件，但它向系统提供块设备能力，因此可以注册为 Device。

同样：

* GPT Partition 可以成为虚拟块设备。
* Framebuffer 可以成为显示设备。
* 虚拟终端可以成为字符设备。

设备抽象层只关心设备提供的能力，而不关心设备是否直接对应硬件。

这里的“虚拟设备”不仅仅是一个抽象概念，还应当在设备树中作为正式节点存在。

例如：

* PCI/Platform/ISA/USB 等总线上的真实硬件设备，是设备树中的基础节点。
* NVMe Namespace、GPT Partition、AHCI Port Disk、ATAPI CDROM、VTTY/PTS 等，应作为这些节点的子节点或派生节点注册进同一棵树。

这意味着：

* 设备树仍然是系统中设备身份与父子关系的唯一真相。
* 虚拟设备不是脱离设备树单独维护的一张表。
* 设备抽象层只是为设备树节点补充统一访问能力，而不是替代设备树。

---

## 与驱动模型的关系

设备抽象层解决的是：

“系统如何统一表示和访问设备。”

驱动解决的是：

“如何控制具体硬件。”

两者关系：

```
Device Abstraction Layer
          |
          v
       Device

          |
          v
    Driver Operations

          |
          v
      Hardware
```

通过该设计，设备管理、设备访问以及硬件实现被分离，使内核可以在不修改上层模块的情况下增加新的设备驱动。

现在内核的 `(v)tty` 也应当最终融入这个框架，但不建议在第一阶段就完成。

---


当前系统已经具有设备树/设备节点管理雏形，因此本设计不应另起一套平行的 `Device` 主模型。

正确方向应为：

* `DeviceNode` 继续负责：
  * 节点身份
  * 父子层级
  * 总线类型
  * 资源描述
  * 驱动绑定与 `driver_data`
* 统一的 `DeviceOps`/`read-send-ctrl` 访问层负责：
  * 统一 I/O 能力入口
  * 将具体访问分发到驱动实现
  * 为 `/dev`、`/.dev`、挂载关系等上层视图提供后端

因此，这一方案是对设备树系统的补强，而不是替代：

* 设备树负责“它是什么、它挂在哪、它从属于谁”
* 统一操作层负责“它现在能怎么被访问”

如果将来实现时把 `Device` 做成与 `DeviceNode` 并列、且分别维护生命周期/父子关系/命名，则会造成两份真相，是应避免的退步。

---


本设计采用两套设备视图，加上一套挂载视图：

## 1. `/dev`：Linux 风格访问视图

`/dev` 的职责是“稳定打开设备”，而不是展示设备拓扑。

适合暴露：

* `/dev/tty`
* `/dev/pts/N`
* `/dev/fd0`
* `/dev/sr0`
* `/dev/nvme0`
* `/dev/nvme0n1`

特点：

* 路径短
* 命名稳定
* 面向上层程序与通用 I/O
* 只有具备 I/O 能力的终端节点才需要出现在这里

## 2. `/.dev`：设备树等价动态映射视图

`/.dev` 的职责是把设备树结构以目录树方式动态映射出来，用于：

* 查看父子关系
* 查看总线归属
* 查看虚拟设备来源
* 调试、枚举、定位设备
* 为 `/dev` 中的稳定别名提供来源追踪

例如，设备树中：

```text
system-root
  pci-root
    pci-bus@0000:00
      isa@0000:00:01.0
        fdc@0
          floppy@0
```

可在 `/.dev` 中动态映射为：

```text
/.dev/system-root/pci-root/pci-bus@0000:00/isa@0000:00:01.0/fdc@0/floppy@0
```

注意：

* `/.dev` 要忠实反映设备树层级。
* `/.dev` 中不是所有节点都必须支持 `open/read/send/ctrl`。
* 总线节点、桥节点、纯管理节点可以只提供枚举/查询意义，而不是可打开的流或块对象。

## 3. `/mnt`：文件系统挂载视图

`/mnt` 的职责仍然是挂载点命名空间，而不是设备树镜像。

但是，`/mnt` 中挂载的每个文件系统都应可关联回 `/.dev` 中的某个存储设备节点。

这意味着：

* `/mnt/data` 不需要长成设备树路径。
* 但系统内部应记录：`/mnt/data` 当前挂载自哪个 `DeviceNode`。
* 这样可以从挂载点反查到底层存储设备。
* 也可以从 `/.dev/.../nvme0n1` 反查它当前被挂载到哪些路径。

因此三者职责划分为：

* `DeviceNode`：设备对象真相
* `/.dev`：设备树动态投影
* `/dev`：稳定设备访问入口
* `/mnt`：文件系统挂载命名空间

---


统一访问接口仍采用最小三元组：

```cpp
struct DeviceOps
{
    int (*f_read)(
        Device *,
        void *,
        size_t,
        idx_t,
        int
    );

    int (*f_send)(
        Device *,
        void *,
        size_t,
        idx_t,
        int
    );

    int (*f_ctrl)(
        Device *,
        int,
        void *,
        int
    );
};
```

其中：

* `f_read`：统一读入口
* `f_send`：统一发送/写入口
* `f_ctrl`：统一控制入口

这里的 `send` 在块设备上通常等价于写，在流设备上更接近顺序输出。

需要注意：

* 这三个接口本身并不等于“Linux 风格设备文件接口”。
* 它们更适合作为驱动后端统一入口。
* 要实现 `open("/dev/xxx")` 这类行为，还需要额外的设备文件包装层、句柄层和 VFS 转发层。

可理解为：

* `read/send/ctrl` 解决“设备能力统一化”
* `open/close/seek` 及 `vfs_file` 包装解决“设备文件化”

---

## Stage 0：建立统一能力层，不改主视图语义

目标：

* 保留 `DeviceNode` 作为唯一设备真相。
* 在 `DeviceNode + driver_data` 基础上补充统一访问能力层。
* 定义哪些节点是“可 I/O 节点”，哪些只是拓扑节点。

本阶段工作：

* 正式抽象统一的 `DeviceOps(read/send/ctrl)`。
* 将统一访问入口挂接到现有 `DeviceNode` 模型之上。
* 不建立独立平行的设备主表。
* 先只服务内核内部调用。
* `/dev` 暂保持现状。
* `/.dev` 先只确定映射规则，不要求立即完整导出。

本阶段重点不是“打开设备文件”，而是先把能力模型立住。

## Stage 1：先接入块设备与存储类虚拟设备

目标：

* 让最稳定、最容易统一语义的设备先接入 `read/send/ctrl`。

优先对象：

* NVMe Controller / Namespace
* AHCI Disk / ATAPI CDROM
* PATA Disk / Floppy
* GPT Partition 等存储类虚拟设备

本阶段工作：

* 让这些设备节点实现统一 `read/send/ctrl`。
* 在设备树中正式注册这些虚拟设备节点。
* 开始建立 `/dev` 稳定命名与 `/.dev` 拓扑路径之间的对应关系。

示例：

* `/.dev/.../floppy@0` <-> `/dev/fd0`
* `/.dev/.../namespace@1` <-> `/dev/nvme0n1`
* `/.dev/.../cdrom@0` <-> `/dev/sr0`

本阶段先不处理复杂 tty 语义。

## Stage 2：导出 `/dev` 与 `/.dev`，并建立 `/mnt` 关联

目标：

* 让统一设备层真正进入 `DevFs/VFS/Fileman` 路径。
* 形成稳定访问视图与拓扑视图双轨并存。

本阶段工作：

* 在 `DevFs` 中导出 Linux 风格 `/dev` 设备入口。
* 在 `DevFs` 或对应视图层中导出 `/.dev` 动态树映射。
* 把 `open/read/write/ioctl` 风格调用转发到 `DeviceNode.ops`。
* 为 mount/superblock 记录底层来源的 `DeviceNode`。

完成后：

* 可通过 `/dev/xxx` 稳定访问设备。
* 可通过 `/.dev/...` 查看设备来源与父子关系。
* 可从 `/mnt/data` 反查到底层 `/.dev` 存储设备。
* 可从 `/.dev/.../nvme0n1` 反查当前挂载点。

## Stage 3：接入 tty / pts / console 等复杂流设备

目标：

* 将最复杂的字符设备语义最后纳入统一框架。

对象包括：

* `/dev/tty`
* `/dev/pts/N`
* 串口设备
* 虚拟终端
* console 相关流设备

本阶段工作：

* 让这些设备最终也通过 `read/send/ctrl` 暴露能力。
* 保留现有阻塞、前台 tty、focus、console 线程等语义。
* 在 `/dev` 中提供稳定 Linux 风格入口。
* 在 `/.dev` 中提供真实拓扑归属或虚拟归属。

这一阶段风险最高，因此故意放到最后。

---


如果按以上四阶段推进，则工程量可控，且能避免一次性大重构。

大致判断：

* Stage 0：不大
* Stage 1：中等
* Stage 2：中到偏大
* Stage 3：偏大，且风险最高

其中最难的通常不是定义 `DeviceOps`，而是：

* 如何把现有 `DevFs/VFS/Fileman` 路径平滑接过去
* 如何让 `/dev` 与 `/.dev` 共用同一套真实后端
* 如何在不破坏 tty 历史语义的情况下纳入统一框架

---


1. 设备树是真相，不能被平行设备表替代。
2. `DeviceOps` 是能力层，不是新的拓扑主模型。
3. `/dev` 面向稳定访问，`/.dev` 面向真实拓扑。
4. `/mnt` 是挂载命名空间，但必须能关联回 `/.dev` 存储设备。
5. 虚拟设备必须进入设备树，而不是单独游离维护。
6. 块设备先行，tty 最后处理。

## 实际实现

stage0 stage1

剩余
- stage2
- stage3
- “每个设备都能定义一组自己的属性，例如 AHCI 显示 port 数、CAP、link，IDE 显示 primary/secondary、compat/native、挂了几个设备”








