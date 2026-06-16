
```
我现在想显示FreeType字体。为了我想：基于  VideoConsole2 实现 VideoConsole3 ，就像 VideoConsole2 基于 VideoConsole 那样（注意不是继承）。使用 VideoConsole3  主要是为了避免影响VideoConsole2 的业务逻辑。

相关文件如下
D:\her\unisym\inc\cpp\Device\_Video.hpp
D:\her\mecocoa\depends\freetype.cpp

现在你不需要指定实施方案，也不要修改代码，只需要评估我的想法是否可行，以及有没有更好的实现
```

```
我现在有2个想法
1、将Vcon2拓展一下。直接在Vcon2上操作，Mecocoa只需要配置好接口
2、基于升级后的Vcon2。在Mecocoa里再实现Vcon3

请评估这两个方案(现在你不需要指定实施方案，也不要修改代码)
```


```
【MCCA 0x8632】自上一次git commit 到现在，我尝试移植FreeType 结合Simsun字体显示中文日文。
现在系统启动的时候会预加载ttf前面40KB左右用来显示ASCII。目前有这些问题
- 现在能够显示ascii，但是我不确定显示的ascii是不是ttf中的字体，也有可能是系统自带的16x8点阵字体
- 完全无法显示非ASCII字符

当前修改过的文件除了本项目，还有D:\her\unisym\lib\cpp\Device\Video-VideoConsole2.cpp

请检查原因(现在你不需要指定实施方案，也不要修改代码)
如果无法确定是什么原因，可以设计插桩方案，但是修改代码需要我的同意
```

```
谢谢！在您的方案下，我确定了我的ASCII确实来自于TTF。那么现在问题就可以缩小了：现在输出非ASCII字符不显示，FreeType也没有触发Lazy加载，大概率是Vcon2没有正确识别Unicode编码导致的。为了解决这个问题，可能需要结合String类（仅供参考）
请为我指定中文的实施方案，待我同意后再实施

约束条件如下
（需求：中文回答 英文注释 TAB缩进 

尽量见到代码字面就知道意思，可以适当加注释说明 

原来存在的注释保留 不要去掉 代码风格参考源代码）

构建让我来，不要擅自make run。

不应该凭经验猜测枚举值，必须以头文件中的定义为唯一准则。

禁止擅自git checkout restore stage
```

```
谢谢！可以显示了。现在的问题
- Win+C 创建新SHell 时会卡死
- 刚刚打印了一个中文，之后再打印需要重新读盘，似乎没有一个缓冲区存放一下
- 显示完一段中文后显示ASCII好慢，光标移动也好卡，不知道是什么原因
请寻找这些问题是否属实以及解决方式，针对这些问题用中文指定一个新的实施方案
```

