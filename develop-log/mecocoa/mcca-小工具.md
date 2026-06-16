
```
请讲 D:\her\mecocoa\configs\atx-x86-flap32.make build_util的应用 转移到 D:\her\mecocoa\subapps\Makefile.gcc.x86，保持 主文件清晰。

并且判断如果二进制最新，就不需要更新二进制了。make clean 需要清除二进制的目录

【询问】D:\her\mecocoa\accmlib\x86\others-a.asm 能不能放一个 wake 声明。call _preprocess。用于初始化C++的Init段。如果程序没有链接到，说明是C程序或其他程序，就不管，否则调用这个函数
```

```
我要为x8632设置新的工具程序： cp rm mv mkdir echo clear
- rm 要求能删除文件、文件夹
- clear 的重点不是工具，而是让 VideoConsole2 支持 \033c
相关源文件都在：D:\her\unisym\demo\utilities

其他小工具可供参考 ： D:\her\mecocoa\subapps\Makefile.gcc.x86，例如其中的 ls pwd 等

建议使用POSIX接口

请先不要动代码，先检查操作系统对这些功能(系统调用 )的支持是否完备
```
