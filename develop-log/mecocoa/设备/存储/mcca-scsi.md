
# 20260727

**使用SAS控制器时，驱动的是控制器本身，而不是直接驱动硬盘**。

在物理连接上，兼容性是**单向**的：
- **SAS控制器 → SATA硬盘：可以**。SAS控制器在设计时就考虑了对SATA硬盘的兼容[](https://baike.baidu.com/item/%e4%b8%b2%e8%a1%8c%e8%bf%9e%e6%8e%a5SCSI%e6%8e%a5%e5%8f%a3%28SAS%29/3065016)。它通过内置的**STP（SATA通道协议）**，将SATA硬盘的ATA指令“翻译”成自己能够管理的格式[](https://www.pconline.com.cn/ask/406630.html)[](https://baike.baidu.com/item/%e4%b8%b2%e8%a1%8c%e8%bf%9e%e6%8e%a5SCSI%e6%8e%a5%e5%8f%a3%28SAS%29/3065016)。
- **SATA控制器 → SAS硬盘：不可以**。SATA控制器无法理解SAS硬盘使用的SCSI指令集[](https://www.pconline.com.cn/ask/367112.html)[](https://www.pconline.com.cn/ask/308810.html)。即便物理上能插上，系统也无法识别[](https://www.pconline.com.cn/ask/367112.html)。
    
所以在硬件层面，**只有SAS控制器能“容纳”SATA硬盘**。

|技术世代|控制器类型 / 协议|主要特点与性能|典型产品或芯片|
|---|---|---|---|
|**并行SCSI**  <br>(已淘汰)|**SCSI-1**[](https://server.it168.com/a2004/0709/8809/000000088098_2.shtml)[](https://baike.baidu.com/item/scsi%E7%A7%8D%E5%9E%8B/2169029)|最早的SCSI标准，5MB/s[](https://server.it168.com/a2004/0709/8809/000000088098_2.shtml)[](https://baike.baidu.com/item/scsi%E7%A7%8D%E5%9E%8B/2169029)。|N/A|
||**SCSI-2 / Fast SCSI**[](https://server.it168.com/a2004/0709/8809/000000088098_2.shtml)[](https://baike.baidu.com/item/scsi%E7%A7%8D%E5%9E%8B/2169029)|10-20MB/s，引入了Wide（16位）总线[](https://server.it168.com/a2004/0709/8809/000000088098_2.shtml)。|N/A|
||**SCSI-3 / Ultra SCSI**[](https://server.it168.com/a2004/0709/8809/000000088098_2.shtml)|20-40MB/s，包含Ultra和Ultra Wide[](https://server.it168.com/a2004/0709/8809/000000088098_2.shtml)。|N/A|
||**Ultra2 SCSI**[](https://server.it168.com/a2004/0709/8809/000000088098_2.shtml)[](https://baike.baidu.com/item/scsi%E7%A7%8D%E5%9E%8B/2169029)|采用LVD技术，速率达40-80MB/s[](https://server.it168.com/a2004/0709/8809/000000088098_2.shtml)。|N/A|
||**Ultra3 / Ultra160 SCSI**[](https://server.it168.com/a2004/0709/8809/000000088098_2.shtml)[](https://baike.baidu.com/item/scsi%E7%A7%8D%E5%9E%8B/2169029)|160MB/s，引入了CRC校验等新功能[](https://server.it168.com/a2004/0709/8809/000000088098_2.shtml)。|LSI53C896、AIC-7899|
||**Ultra320 SCSI**[](https://server.it168.com/a2004/0709/8809/000000088098_2.shtml)|并行SCSI的终极形态，速率达320MB/s。|LSI1032E70LT|
|**串行SCSI**  <br>(现代主流)|**SAS** (Serial Attached SCSI)|点对点串行连接，向下兼容SATA[](https://baike.baidu.com/item/SAS%E6%8E%A7%E5%88%B6%E5%99%A8)，理论可连接超16000个设备[](https://baike.baidu.com/item/SAS%E6%8E%A7%E5%88%B6%E5%99%A8)。|LSI Logic SAS[](https://techdocs.broadcom.com/tw/zh-tw/vmware-cis/vsphere/vsphere/6-7/vsphere-virtual-machine-administration-guide-6-7/configuring-virtual-machine-hardware-vSphereVirtualMachineAdministration/scsi-controller-configuration-vSphereVirtualMachineAdministration.html)、Adaptec SAS 5805、Broadcom SAS 3008|
|**虚拟化/专用**  <br>(软件层)|**VMware 准虚拟 SCSI** (PVSCSI)|专为虚拟机优化，能提升吞吐量并降低CPU负载[](https://techdocs.broadcom.com/tw/zh-tw/vmware-cis/vsphere/vsphere/6-7/vsphere-virtual-machine-administration-guide-6-7/configuring-virtual-machine-hardware-vSphereVirtualMachineAdministration/scsi-controller-configuration-vSphereVirtualMachineAdministration.html)[](https://techdocs.broadcom.com/cn/zh-cn/vmware-cis/vsphere/vsphere/7-0/scsi-controller-configuration.html#GUID-5872D173-A076-42FE-8D0B-9DB0EB0E7362-en)。|VMware虚拟控制器[](https://techdocs.broadcom.com/tw/zh-tw/vmware-cis/vsphere/vsphere/6-7/vsphere-virtual-machine-administration-guide-6-7/configuring-virtual-machine-hardware-vSphereVirtualMachineAdministration/scsi-controller-configuration-vSphereVirtualMachineAdministration.html)[](https://techdocs.broadcom.com/cn/zh-cn/vmware-cis/vsphere/vsphere/7-0/scsi-controller-configuration.html#GUID-5872D173-A076-42FE-8D0B-9DB0EB0E7362-en)|
|**SAN存储协议**  <br>(网络层)|**光纤通道** (FC)|通过光纤网络传输SCSI指令（FCP），用于构建大型存储网络（SAN）。|Emulex Lpe12002、QLogic QLE2870|
||**iSCSI**|基于TCP/IP网络传输SCSI指令，成本较低。|软件发起端或硬件TOE卡|

---

剩下的工作还有，但都属于“下一阶段增强”，不该再卡这次提交：

- 更完整的 REQUEST SENSE / 错误恢复收尾
- 更多 target / lun 组合验证
- 无盘、换盘、NOT READY、UNIT ATTENTION 等异常路径
- 进一步清理 scsidisk.cpp 里的 bring-up 代码和日志









