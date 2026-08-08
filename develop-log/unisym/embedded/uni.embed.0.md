
# 20260722

```
Unisym 库现在有HAL库，在 D:\her\unisym\inc\cpp\Device\*

对标的内容是
- D:\tmp\__hal_origin\Stm32F1\STM32F1xx_HAL_Driver
	- 里面有 git 仓库。你可以阅读，删掉的都是 unisym 适配的内容。但是我不保证适配的内容是没有问题的
- D:\tmp\__hal_origin\Stm32F4
	- 这个没有git仓库，这个目录里的stm32 hal内容都是完全的
- D:\tmp\__hal_origin\Stm32H7\
- D:\tmp\__hal_origin\Stm32Mp13

一些用户示例位于 D:\her\alicewiki\tutorial\mcudev\demo 仅供参考

我现在要研究 GPIO 这一个外设。

请分析
- unisym设备接口是否合理
- 从对标的内容迁移过来的代码逻辑有没有问题
- 在不同优化等级下有没有问题，尤其是 O2 O3 Os

在分析后总结以上问题。并总结该外设的接口设计使用方法。最好给出相关的用户实例使用例程。

【禁止修改代码。仅分析】
```

```
Unisym 库现在有HAL库，在 D:\her\unisym\inc\cpp\Device\*

对标的内容是
- D:\tmp\__hal_origin\Stm32F1\STM32F1xx_HAL_Driver
	- 里面有 git 仓库。你可以阅读，删掉的都是 unisym 适配的内容。但是我不保证适配的内容是没有问题的
- D:\tmp\__hal_origin\Stm32F4
	- 这个没有git仓库，这个目录里的stm32 hal内容都是完全的
- D:\tmp\__hal_origin\Stm32H7\
- D:\tmp\__hal_origin\Stm32Mp13

一些用户示例位于 D:\her\alicewiki\tutorial\mcudev\demo 仅供参考

我现在要研究 UART 这一个外设。

我朋友的分析与总结报告在 C:\Users\phina\Downloads\doc-qrs\ori-xart.md

请分析我朋友的报告是否正确（逐句分析）。你可能需要使用 git log/diff 来这些仓库逐个检查。然后告诉我我朋友报告中有问题的地方。没有问题的地方就不需要提了。


【禁止修改代码。仅分析】
```

```
请再次检查 F1 F4 H7 MP13 的 UART USART 是否都适配了？
且unisym是否能替换 D:\tmp\__hal_origin 里面所有的 *hal_u(s)art.c/h ？

模仿C:\Users\phina\Downloads\doc-qrs\ori-gpio.md，在 ori-xart.md 再追加一个章节：
内容是
一、
请再次检查 F1 F4 H7 MP13 的 UART USART 是否都适配了？即unisym是否能替换 D:\tmp\__hal_origin 里面所有的 *hal_u(s)art.c/h ？如果不能替换，针对每个平台总结替换对应平台的 *hal_u(s)art.c/h 需要修补的部分记录在这一章
二、
当前没有实现的内容。


请检查 F1 F4 H7 MP13 的 GPIO 是否都适配了？即unisym是否能替换 D:\tmp\__hal_origin 里面剩余的所有的 *hal_gpio(_ex).c/h ？
```

```
...
我现在要研究 GPIO 这一个外设。

现在：HAL 的 DeInit 不只是把 pin 改成输入，它还会恢复 MODER/OTYPER/OSPEEDR/PUPDR/AFR，并清 EXTI 映射、IMR/EMR、RTSR/FTSR。Unisym 现在没有 `DeInit()` 或 `Reset()` 级别的公开接口。

我决定在 D:\her\unisym\inc\cpp\Device\GPIO 的 GPIN 的 setMode 方法之后增加一个 bool canMode() 方法，实现DeInit 

各个平台的实现逻辑参考HAL库
```

```
让我们先研究H7。

D:\tmp\__hal_origin\Stm32H7\STM32H7xx_HAL_Driver\Src\stm32h7xx_hal_usart.c D:\tmp\__hal_origin\Stm32H7\STM32H7xx_HAL_Driver\Inc\stm32h7xx_hal_usart_ex.h D:\tmp\__hal_origin\Stm32H7\STM32H7xx_HAL_Driver\Inc\stm32h7xx_hal_usart.h  D:\tmp\__hal_origin\Stm32H7\STM32H7xx_HAL_Driver\Src\stm32h7xx_hal_uart.c D:\tmp\__hal_origin\Stm32H7\STM32H7xx_HAL_Driver\Src\stm32h7xx_hal_uart_ex.c D:\tmp\__hal_origin\Stm32H7\STM32H7xx_HAL_Driver\Inc\stm32h7xx_hal_uart.h D:\tmp\__hal_origin\Stm32H7\STM32H7xx_HAL_Driver\Inc\stm32h7xx_hal_uart_ex.h

看看这些文件的函数有哪些功能是unisym h7里面不具备的，列出来。我计划用unisym 取代官方HAL
```

20260803
```
我不建议新增函数 像这样，我建议新建一个参数或者状态。例如：
新增参数method，method 是 loop, rupt, DMA 等枚举值之一
out(const char* str, stduint len, Method method) // 是虚函数，需要重载新方法
stduint Receive(char* rx_data, stduint size,  Method method = loop);// 这个不是虚函数，所以可以直接新增参数
这个想法怎么样
```

```
告诉你一件事情，以后记住就好。如果在编辑文件的时候发现行末LF CRLF 不一致的时候。请统一。改成什么看首行的标记。
```

```
我现在想优先实现 H7 你看看 H7 和 F4 的相似程度怎么样，如果高的话我会考虑同时实现这两个平台
```
