
- [ ] IDE/PATA硬盘访问速度极慢，其他 IDE CD、SATA等访问都很快 -- 20260719 found -- 

```
既然您确认了没有发生 fallback to PIO（即 DMA 实际上是成功的），那么 IDE/PATA 硬盘极其缓慢的元凶就完全在于过小的 DMA 传输粒度与极度高昂的上下文切换（中断和消息传递）开销结合在了一起。

具体分析如下：

1. 单次 DMA 传输粒度被硬编码为仅 512 字节
在 hd_read_dma_once 和 hd_write_dma_once 中，无论上层请求多大的数据，PRDT（物理区域描述符表）都被硬编码限制在了单扇区：

cmd.count = 1; 
// ...
state.prdt[0].byte_count = 512;
这导致 DMA 失去了它本应有的“大块数据批量搬运”的优势，退化成了每次只搬运 512 字节。

2. 每 512 字节都要经历一次完整的中断挂起与唤醒（最致命的性能瓶颈）
在启动 DMA 后，代码调用了 hd.fn_int_wait()，该指针指向了 hd_int_wait()：

static bool hd_int_wait() {
    CommMsg msg;
    sysrecv(INTRUPT, (&msg), 0); // 阻塞当前任务，等待硬件中断消息
    return true;
}
这意味着：每读取仅仅 512 字节，当前任务就会调用 sysrecv 挂起自己，引发一次操作系统的任务上下文切换。 硬件虽然在一瞬间就传完了这 512 字节并触发了硬件中断 Handint_HDD()，但中断函数需要通过微内核消息机制 rupt_proc 发送消息，调度器再把服务任务唤醒，再次引发上下文切换。 由于操作系统线程调度的延迟（通常在微秒甚至毫秒级）远大于硬盘 DMA 传输 512 字节的时间，你的读写性能实际上被系统调度的瓶颈卡死了。比如读取 1MB 数据，系统要强行休眠和唤醒 2048 次。

3. 为什么其他设备（SATA、IDE CD）访问很快？
SATA (AHCI 驱动)： 通过查看 AHCI 驱动的实现（Harddisk_SATA_AHCI::ReadSectors），可以发现 AHCI 等待命令完成时使用的是轮询 (Polling)：

while (port.tfd & (0x80u | 0x08u)) {} // 直接在 CPU 上自旋等待
port.ci = 1u;
if (!PollCommandSlot(1u)) return false;

它没有调用 sysrecv 休眠线程，避免了数千次的上下文切换开销，完全发挥了硬件的速度。
IDE CD (ATAPI 光驱)： 光驱的扇区大小（Block_Size）是 2048 字节。而在 hd_read_prefer_dma 中有这样一个判断：if (hd.Block_Size == 512) 才走 DMA。因为不满足条件，IDE CD 彻底绕开了这套带有调度瓶颈的单扇区 DMA 逻辑，直接使用了底层的原生 Read()。如果底层 Read()（PIO 模式）是直接通过 rep insw 并在原地死循环轮询 DRQ，虽然耗费 CPU，但它没有休眠唤醒的延迟，因此实际上比这种“慢速 DMA”要快得多。
总结：PATA 速度慢是因为实现上采用了 “单扇区 DMA + 同步阻塞挂起” 的设计。要提升速度，需要通过扩充 PRDT 来支持多扇区连续 DMA 传输，以平摊中断和任务切换的开销。
```

