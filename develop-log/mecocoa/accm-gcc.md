
# 20260528

阶段 1：x86_64-mcca-elf-gcc
  用来编译 kernel、boot code、freestanding runtime

阶段 2：mcca-sysroot + libc + crt0
  能编用户态静态程序

阶段 3：动态链接器 + ABI note
  能编 .so / 动态程序

阶段 4：patch GCC/binutils
  正式变成 x86_64-mcca-gcc

gcc-1.16.1
binutils-2.44

### 先做“能编内核/裸机程序”的版本

```
~/cross/
  src/
    gcc-16.1.0/
    binutils-2.44/

  build-binutils/
  build-gcc/

  install/
```

```bash
export TARGET=x86_64-mcca-elf
export PREFIX=$HOME/opt/cross
export PATH="$PREFIX/bin:$PATH"

# binutils
mkdir build-binutils && cd build-binutils

../src/binutils-*/configure \
  --target=$TARGET \
  --prefix=$PREFIX \
  --with-sysroot \
  --disable-nls \
  --disable-werror \
  --enable-multilib
  
make -j$(nproc)
make install
cd ..

# gcc，仅 C/C++，不依赖 libc
mkdir build-gcc && cd build-gcc
../src/gcc-*/configure \
  --target=$TARGET \
  --prefix=$PREFIX \
  --disable-nls \
  --enable-languages=c,c++ \
  --without-headers \
  --disable-hosted-libstdcxx \
  --disable-shared \
  --disable-threads \
  --disable-libssp \
  --disable-libquadmath \
  --disable-libgomp \
  --disable-libatomic \
  --enable-multilib 
  
#--with-multilib-list=m32,m64
make -j$(nproc) all-gcc all-target-libgcc
make install-gcc install-target-libgcc
```

**sysroot 默认路径**：
- GCC 内置的 sysroot 是在 configure 时通过 `--with-sysroot` 指定的
- 你当时构建时用的 sysroot 路径会被写进 GCC 内部 specs 文件
- 运行时，如果你复制 GCC 给别人，GCC 会默认去 **它构建时指定的 sysroot 路径** 找头文件和库

这样你会得到：
x86_64-mcca-elf-gcc
x86_64-mcca-elf-ld
x86_64-mcca-elf-as

---

现在：TARGET=x86_64-mcca-elf
先别急着：x86_64-mcca
因为：GCC 不认识 mcca OS
而：-elf 天然就是：freestanding target
最适合 OS 开发前期。

### 第二步 静态ELF32/64库

已经有：  
x86_64-mcca-elf-gcc  
  
现在要做到：  
- 能编译用户态静态 ELF 程序  
- 有最小 libc  
- 有 sysroot  
- 有 crt0  
- 能运行 hello world

```
export TARGET=x86_64-mcca-elf
export PREFIX=$HOME/opt/cross
export PATH="$PREFIX/bin:$PATH"

export SYSROOT=/her/mecocoa/accmlib/sysroot
```

现状
```
现在我的结构：

D:\her\mecocoa\accmlib>tree /f
Folder PATH listing for volume Slaver
Volume serial number is 0C35-15B6
D:.
│   accmrv.ld
│   accmrv32.make
│   accmrv64.make
│   accmx64.make
│   accmx86.make
│   lib.cpp
│   lib_con.cpp
│   lib_fileman.cpp
│   lib_memory.cpp
│   lib_process.cpp
│   Makefile
│   README.md
│
├───inc
│       aaaaa.h
│       posix.h
│
...
│
├───riscv
│       others-a.S
│
├───sysroot
│   ├───lib
│   │   ├───aarch64-mcca
│   │   └───x86_64-mcca
│   └───usr
│       └───lib
│           ├───aarch64-mcca
│           └───x86_64-mcca
├───x64
│       others-a.asm
│
└───x86
        others-a.asm

D:\her\unisym\inc\c\ISO_IEC_STD>ls
assert.h    errno.h     inttypes.h  locale.h    signal.h    stdint.h    string.h    wchar.h
complex.h   fenv.h      iso-iec.h   math.h      stdarg.h    stdio.h     tgmath.h    wctype.h
ctype.h     float.h     iso646.h    setjmp.h    stdbool.h   stdlib.h    time.h

D:\her\unisym\inc\c\API-POSIX>tree /f .
Folder PATH listing for volume Slaver
Volume serial number is 0C35-15B6
D:\HER\UNISYM\INC\C\API-POSIX
│   api_posix.h
│   dirent.h
│   fcntl.h
│   unistd.h
│
└───sys
        mman.h
        stat.h
        wait.h 
```

sysroot 是安装目录，不是 build 目录；普通 `.o` 留在 build 里，只有 `crt*.o` 和库文件安装进 sysroot。
当你用 `--sysroot=$SYSROOT` 时，GCC 会自动查找：（crt0.o, libc.a）
```
$SYSROOT/lib/<triple>/
$SYSROOT/usr/lib/<triple>/
```

也就是说，`sysroot` 里可以只放：

```
sysroot/
  usr/lib/x86_64-mcca/
    crt0.o
    libc.a
```

源码可以放在 usr/lib 或 lib 吗？
- 技术上可以，但不建议把源码放进 sysroot/usr/lib 或 sysroot/lib。

暂时不用：

```
sysroot/lib/<triple>/
```

因为你现在还没有动态链接器，也没有“系统启动必须的共享库”这个需求。

e.g.
```
$TARGET-g++ \
  --sysroot="$SYSROOT" \
  -ffreestanding \
  -fno-exceptions \
  -fno-rtti \
  -nostdinc \
  -nostdinc++ \
  -isystem "$SYSROOT/usr/include" \
  -c xxx.cpp \
  -o xxx.o
```


---

工具链 **x86_64-mcca-elf-gcc** 默认只生成 64 位代码，也只包含 **64 位 libgcc**，所以在尝试编译 32 位程序（`-m32`）时会出现这些错误：

```
skipping incompatible .../libgcc.a when searching for -lgcccannot find -lgcc
```

原因：
1. 你的 gcc 没有开启 **multilib**，所以它没有 32 位版本的 `libgcc.a`。
2. `-m32` 只是告诉编译器生成 32 位代码，但链接器找不到对应的 32 位运行时库，所以报错。
3. 你的 `crt0.o` 和 libc.a 也都是 32 位，但放在 64 位的 sysroot 目录下，进一步导致架构不匹配。

```
export TARGET=i686-mcca-elf
export PREFIX=$HOME/opt/cross-i686
export PATH="$PREFIX/bin:$PATH"
```




