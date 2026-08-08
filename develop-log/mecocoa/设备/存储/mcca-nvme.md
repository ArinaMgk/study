# 20260730

## 当前已实现

- PCI NVMe 控制器探测与 BAR0 MMIO 绑定
- Admin Queue 初始化与控制器启停
- Identify Controller
- NVMe 1.0 控制器兼容路径
  - 对不支持 `Identify Namespace List (CNS=2)` 的控制器，回退到 `NN` 枚举 namespace
- I/O Queue 初始化
- Namespace 枚举与 Identify Namespace
- `Harddisk_NVMe` 块设备接入
- 设备树注册
  - 节点名形如 `nvme-disk@0n1`
- 分区解析
  - 统一复用现有公共分区链，支持 `MBR` 与 `GPT`
- 文件系统挂载
  - 挂载路径形如 `/mnt/nvme0n1.P`
- 常规读写路径
  - x86 可正常工作
  - x64 可正常工作
- PRP 描述增强
  - 单页传输使用 `PRP1`
  - 两页传输使用 `PRP1 + PRP2`
  - 多页传输可使用 `PRP list`
- MSI 单向量中断路径
  - 固定向量 MSI 配置
  - I/O completion IRQ 入口与完成队列认领
  - 中断聚合关闭

## 当前已验证

- x86 下 NVMe 主路径可正常工作
- x64 下 NVMe 主路径可正常工作
- 控制器探测、Identify、namespace 枚举正常
- 分区解析与 FAT 挂载正常
- 常规读写正常

## 已实现但暂未充分测试

- `PRP2 / PRP list` 覆盖的 4K 以上连续大块 I/O
- 更大文件场景下的连续读写压力

## 当前未做或未完成

- MSI-X / 多向量中断路径
- 更完整的 flush / reset / 错误恢复
- 多控制器场景验证
- 更系统的性能与压力测试
