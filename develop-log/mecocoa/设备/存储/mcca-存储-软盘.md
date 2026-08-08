
# 20260517

## 1. 检测有没有软盘驱动器、有几个？

在 PC 架构中，系统启动时 BIOS 会将检测到的硬件信息存储在 **CMOS** 中。我们可以通过读取 CMOS 的 `0x10` 寄存器来获取软盘驱动器的数量和类型。

### CMOS 寄存器 `0x10` 结构

该寄存器是一个字节（8位）：

- **高 4 位 (Bit 7-4)**：驱动器 A（Drive 0）的类型。
    
- **低 4 位 (Bit 3-0)**：驱动器 B（Drive 1）的类型。
    

类型对应的数值定义如下：

- `0x0`：无驱动器 (None)
    
- `0x1`：360 KB 5.25寸驱动器
    
- `0x2`：1.2 MB 5.25寸驱动器
    
- `0x3`：720 KB 3.5寸驱动器
    
- `0x4`：**1.44 MB 3.5寸驱动器（最常见）**
    
- `0x5`：2.88 MB 3.5寸驱动器
    

### 代码实现

结合你之前的 `uni` 命名空间风格，可以编写如下检测函数：

C++

```
struct FloppyInfo {
    bool has_drive_a;
    bool has_drive_b;
    byte type_a;
    byte type_b;
    int count;
};

FloppyInfo DetectFloppyDrives() {
    FloppyInfo info = {false, false, 0, 0, 0};

    // 访问 CMOS 寄存器 0x10
    // 0x70 是索引端口，0x71 是数据端口
    outpb(0x70, 0x10); 
    byte cmos_val = innpb(0x71);

    info.type_a = cmos_val >> 4;   // 高4位
    info.type_b = cmos_val & 0x0F; // 低4位

    if (info.type_a != 0) {
        info.has_drive_a = true;
        info.count++;
    }
    if (info.type_b != 0) {
        info.has_drive_b = true;
        info.count++;
    }

    return info;
}
```

---

## 2. 检测驱动器里有没有软盘？

检测“有没有插软盘”不能单靠 CMOS，必须实时和软盘控制器（FDC）交互。核心在于利用 **DIR 寄存器（Digital Input Register, 端口 `0x3F7`）** 的 **Disk Change (DKC) 位（Bit 7）**。

### 检测原理

1. **Disk Change 位 (Bit 7)**：如果该位为 `1`，表示驱动器门被打开过、软盘被拔出过，或者**当前根本没有软盘**。
    
2. **清除状态**：当插入一张新软盘时，这个 Bit 7 **不会**自动变回 `0`。你必须通过让磁头**寻道（Seek）**或者**重新校准（Recalibrate）**（即让马达转动并移动磁头改变磁道），FDC 才会去重新检测盘片。
    
3. **最终判断**：如果执行了寻道操作后，Bit 7 依然是 `1`，说明驱动器里**确实没有软盘**。
    

### 代码实现

在上一版本的 `FloppyDisk` 类基础上，可以扩展出 `IsMediaPresent()` 函数：

C++

```
bool FloppyDisk::IsMediaPresent() {
    // 1. 首先确保马达是开启的
    Motor(true);

    // 2. 读取一次数字输入寄存器 (DIR)
    // 如果 Bit 7 是 0，说明软盘在里面且完好，直接返回 true
    if ((innpb(PORT_FDC_DIR) & 0x80) == 0) {
        return true;
    }

    // 3. 如果 Bit 7 是 1，可能是“刚换了盘”或者“没盘”
    // 我们需要强制让磁头动一下（比如重新校准 Recalibrate 或 Seek 磁道）来刷新状态
    WriteCmd(FDC_CMD_RECALIBRATE);
    WriteCmd(id);
    
    if (fn_int_wait) fn_int_wait(); // 等待中断完成

    byte st0, cyl;
    SenseInt(st0, cyl); // 清除中断状态

    // 4. 再次读取 DIR 寄存器
    byte dir = innpb(PORT_FDC_DIR);
    
    Motor(false); // 关闭马达

    // 如果执行完校准动作后，Bit 7 依然是 1，说明没有盘（或者盘坏了/门没关）
    if (dir & 0x80) {
        return false; 
    }

    return true; // 状态被清除了，说明有软盘在里面
}
```

---

## 总结：完整的检测工作流

如果你在写操作系统内核的设备初始化阶段，标准的工作流应该是这样的：

1. **第一步**：读 CMOS `0x10`。如果是 `0`，说明压根没这硬件，直接跳过。
    
2. **第二步**：如果 CMOS 显示存在驱动器，实例化你的 `FloppyDisk` 驱动对象（例如 `FloppyDisk fda(0);`）。
    
3. **第三步**：调用 `fda.Reset()` 初始化控制器。
    
4. **第四步**：当用户尝试挂载 `A:` 盘或读取数据时，调用 `IsMediaPresent()`。若返回 `false`，向用户报错误码：`No Media Present`（请插入软盘）。

# 20260618

给 hd_int_wait() 增加超时机制，超时后回退到轮询模式（ ReactType::Loop ），轮询5次之后就返回错误(只在本次有效)。应用给磁盘和软盘

# 20260716

```
在我的操作系统中 x86/64 目前

| `pci@0000:00:07.0` | `8086` (Intel)    | `7110`     | **Intel 82371AB PIIX4 ISA 桥**             |
| `pci@0000:00:01.0` | `8086` (Intel) | `7000`     | **Intel 82371SB PIIX3 ISA 桥**             |


这两种桥还没有实现、识别。所以在设备树中只是匿名设备。我想实现，展开其中的 isa 设备，请看看要怎么做

注意，目前软盘好像已经在使用 ISA DMA 了

我已经预先留出了对应的代码文件
D:\her\unisym\inc\cpp\Device\Bus\ISA.hpp
D:\her\unisym\lib\cpp\Device\Bus\ISA.cpp
```


