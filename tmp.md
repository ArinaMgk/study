- Intel 64 and IA-32 Architecture Software Developer's Manual (Full Volume Bundle)
- ACPI Specification (version 6.4)
- Devicetree Specification
- ARM® Generic Interrupt Controller (v3)
- Arm® Architecture Reference Manual (Profile-A)
- Procedure Call Standard for the Arm® 64-bit Architecture (AArch64)
- IBM PC/AT Technical Reference
- IBM VGA/XGA Technical Reference
- 82093AA I/O Advanced Programmable Controller (IOAPIC) (Datasheet)
- MC146818A (Datasheet)
- Intel 500 Series Chipset Family Platform Controller Hub (Datasheet - Volume 2)
- PCI Local Bus Specification, Revision 3.0
- PCI Express Base Specification, Revision 1.1
- PCI Firmware Specification, Revision 3.0
- Serial ATA - Advanced Host Controller Interface (AHCI), Revision 1.3.1
- Serial ATA: High Speed Serialized AT Attachment, Revision 3.2
- SCSI Command Reference Manual
- ATA/ATAPI Command Set - 3 (ACS-3)
- ECMA-119 (ISO9660)
- Rock Ridge Interchange Protocol (RRIP: IEEE P1282)
- System Use Sharing Protocol (SUSP: IEEE P1281)
- Tool Interface Standard (TIS) Portable Formats Specification (Version 1.1)
- _Computer System - A Programmer's Perspective Third Edition_ (Bryant, R & O'Hallaron, D), a.k.a. CS:APP
- _Modern Operating System_ (Tanenbaum, A)
- Free VGA, [http://www.osdever.net/FreeVGA/home.htm](http://www.osdever.net/FreeVGA/home.htm)
- GNU CC & LD online documentation.
- PCI Lookup, [https://www.pcilookup.com/](https://www.pcilookup.com/)
- Linux man pages


RSDP-MADT(APIC)/XSDT-

### ## 阶段三：系统拓扑解析 (ACPI)

现代内核不依靠硬编码或探测来猜测硬件，而是读取 ACPI（高级配置与电源接口）表。

- **定位 RSDP (Root System Description Pointer)：** 从 UEFI 传入的系统表中找到 RSDP，进而解析 RSDT 或 XSDT。
    
- **解析 MADT (Multiple APIC Description Table)：** 这是多核启动的关键。从 MADT 中获取：
    
    - 所有的 CPU 核心信息（各个核心的 Local APIC ID）。
        
    - **I/O APIC** 的基地址（负责将外部设备硬件中断路由到各个 CPU 核心）。
        
- **解析 MCFG 等其他表：** 获取 PCIe 的 ECAM（增强配置访问机制）基地址，为后续总线扫描做准备。
    


### ## 阶段五：设备与总线检测 (PCIe Discovery)

在核心和中断就绪后，内核需要找到连接在系统上的外设。

- **PCIe 枚举：** 现代系统使用内存映射的方式（MMIO）来访问 PCI 配置空间，即通过 ACPI MCFG 表提供的 **ECAM** 机制。
    
    - 遍历总线 (Bus)、设备 (Device)、功能 (Function) —— 即 BDF 扫描。
        
    - 读取 Vendor ID 和 Device ID，识别网卡、NVMe 硬盘、USB 控制器和显卡 (Framebuffer) 等。
        
- **中断路由 (MSI/MSI-X)：** 为现代 PCIe 设备配置消息信号中断（Message Signaled Interrupts）。这允许设备直接向指定的 Local APIC 写入数据来触发中断，完全绕过了传统的引脚和 I/O APIC 路由。
    

### ## 阶段六：文件系统、驱动加载与进入用户态

- **VFS 与驱动初始化：** 挂载虚拟文件系统（VFS），加载必要的驱动程序（通常是从 initrd 内存盘中读取）。
    
- **调度器启动：** 启动任务调度器，所有核心开始根据时间片轮转执行任务。
    
- **Ring 3 过渡：** 创建系统中的第一个用户态进程（通常称为 `init`），设置 Ring 3 的段寄存器和用户栈，通过 `iretq` 或 `sysret` 指令从 Ring 0（内核态）降级并跳转到用户态应用程序。至此，内核的启动流程正式完成。





