
# 20260424

```
现在x86能实现fork，我想给x64也实现fork。
相关文件：
-D:\her\unisym\inc\c\proctrl\IAx86_64.h
-D:\her\mecocoa\mecocoa\syscall.cpp
-D:\her\mecocoa\prehost\atx-x64-uefi64\atx-x64.asm
-D:\her\mecocoa\prehost\atx-x86-flap32\atx-x86.asm
-D:\her\mecocoa\mecocoa\taskman-new.cpp

现有设计的fork需要程序syscall前的状态，因此需要给syscall函数传入CallgateFrame* frame。因此是不是需要把原来的Table索引转换为 Handint_SYSCALL。
如果有更好的方案，请提给我。
（先不要改动代码，告诉我方案）

```

```
我的选择：方案 C（类似方案A）
你可以稍微改动CallgateFrame ，但不要影响x86原本 的CallgateFrame 

我的回答
- 当前 x64 是否有 Ring3 进程在用 syscall？ ：有的，原理上Ring1和Ring2也能用fork，最好Ring0也能用。
- CallgateFrame 中 sp 字段：你可以参考D:\her\mecocoa\mecocoa\taskman-new.cpp中的Fork只用了 sp0，sp现在并不重要，sp0是syscall自动压栈的吧？
- 你是否接受每次 syscall 都走 Handint_SYSCALL 而不是直接 table 分发？：这是注定的，每次 syscall 都走 Handint_SYSCALL 更加安全，可以有更复杂的处理和安全判断（如越界处理）

（需求：中文回答 英文注释 TAB缩进 
尽量见到代码字面就知道意思，可以适当加注释说明 
原来存在的注释保留 不要去掉 ）

验证：不要擅自make，现在是在windows环境会失败，我需要去另一台设备环境上验证，我会将运行结果告诉你。
```

```
现有方案需要进行修改：
- 对于x64, CallgateFrame不需要保存 ss0,cs 以及各种段选择子。SS在 SYSRET时用到的是STAR寄存器设置好的，其他段选择子也用不上。可以考虑不压入或者压入0值
- PG_PUSH 会往栈里压入几个数值，要保持等待 PG_POP利用，你注意到了吗
- 考虑刚进syscall在用户栈上压入需要保存的信息（就像我一开始的实现一样），传入的frame指针指向用户非特权栈（处于用户内存空间，用的时候转换一下就可以）
```

