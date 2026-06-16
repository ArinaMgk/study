
```
_MCCA 0x8632：现在内核还不支持环境变量
现在想给 ProcessBlock 实现环境变量，为每个非ring0任务添加默认常用的环境变量：?, PATH(设置两个默认路径”/md0“ 和 ”/mnt34/apps“，即”/md0:/mnt34/apps“), USER(一律用root)
验证1： D:\her\unisym\demo\utilities\args.c
验证2：D:\her\COTLAB\src\cotlab.cpp （需设置 $?，且只有PATH内的a 才能 a 直接执行，否则需要”./a“）
```

```
#### 方案 A：纯用户态维护（传统 POSIX 风格）
注意环境变量具有继承性
采用Linux设计，PATH查找和$?的更新逻辑，都运行在用户程序。

使用中文制定实施方案，并加入以下约束，等待我同意后再执行

（需求：中文回答 英文注释 TAB缩进 
尽量见到代码字面就知道意思，可以适当加注释说明 
原来存在的注释保留 不要去掉 代码风格参考源代码）
构建让我来，不要擅自make run。
不应该凭经验猜测枚举值，必须以头文件中的定义为唯一准则。
禁止擅自git checkout restore stage
```

