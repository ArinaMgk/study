
# 20260617

```
我的想法是默认进入 shell mode。-c -s -f 加载的都是 shell 脚本。argv 带 -m 进入函数模式，-c -s -f 加载的都是类似python那样的脚本。在 shell 模式中通过输入 "cot+mode 1"进入算术模式。在算数模式按下 ESC 或者输入 "." 进入shell 模式。
我的思路是先将 Linux 和 MCCA 分开。先实现MCCA的Shell支持，再慢慢将Linux内容移到MCCA。
```

