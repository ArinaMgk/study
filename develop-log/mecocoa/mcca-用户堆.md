
# 20260523

```
根据我的代码，请看看我的这个方案怎么样

现在我要实现用户栈（第一阶段：匿名映射）
【内核端】
实现系统调用（addr,lin都按照4K对齐）
- MMAP(len, flag, fd)->addr  flag添加页面属性 ，fd用于文件映射，本阶段不考虑
- UMAP(addr,len)->len 返回成功释放的虚拟区域大小，如果涉及文件的映射则len不起作用

[如果使用了内核的页表，则mmap umap 拒绝操作!]
ProcessBlock 的 heaptop  heapbtm 统计堆(页)的地址范围
- heapbtm ，最低地址，一般是 ELF 文件的最高地址加上 0x10000 然后 4K 对齐
- heaptop，4K对齐，最高地址，MMAP的空间越多，值越大
heap 在虚拟空间上是连续的

【用户端】
实现 malloc free，结合 mempool，参考D:\her\mecocoa\mecocoa\memoman.cpp（不需要考虑加锁）

【注意】
1、需要在程序销毁时释放堆空间
2、适配fork等函数
3、MMAP时并不在页表上加上，缺页时再申请
4、注意使用 invlpg

（第二阶段：补全实现文件映射）

现在你不需要指定实施方案，也不要修改代码，只需要评估我的想法是否可行，以及有没有更好的实现
```

```
请用中文制定实施方案（不要动代码）
在 Page Fault 中分配新的物理页后，必须将该物理页清零（Zero-fill）。维护一个 VMA 链表。加锁。
```















