---
author: Jack_Wang
email: 56127817+Outsider565@users.noreply.github.com
title: ICS-linking-hw2
date: 2021-03-29 22:48:51
category: [ICS]
tags: [王少文]
index_img: https://img2.baidu.com/it/u=3459720424,2125514627&fm=26&fmt=auto&gp=0.jpg
---
# ICS作业-链接2参考

## 7.10

A.

```shell
g++ p.o libx.a
```

B.

```shell
g++ p.o libx.a liby.a libx.a
```

C.

```shell
g++ p.o libx.a liby.a libx.a libz.a
```

## 7.11

因为有.BSS区域，该区域在可执行文件中并不分配空间；而在加载入内存时进行分配，因此要预留出0x230字节。

## 7.12

A.

$*refptr = ADDR(r.symbol)+r.addend-refaddr$

$=0x4004f8-4-(0x4004e0+0xa)=0xa$

B.

$*refptr = ADDR(r.symbol)+r.addend-refaddr$

$=0x400500-4-(0x4004d0+0xa)=0x22$

## 7.13

### A.

#### 寻找libc.a和libm.a

使用如下的命令，找到库的位置

```shell
gcc --print-file-name=libc.a
gcc --print-file-name=libm.a
```

得出这两个文件均在`/usr/lib/aarch64-linux-gnu`目录下

#### 查找有多少个Object Files

```shell
cd /usr/lib/aarch64-linux-gnu
ar -t libc.a | wc -l
ar -t libm.a | wc -l
```

得出结论，在libc.a下有1616个Object file，在libm.a下有576个Object file

### B.

他们产生的二进制文件是不同的，-g会携带更多的debug和符号信息。

使用如下命令即可看出他们之间的差别

```shell
	xxd main>main.binary
	xxd maing>maing.binary
	colordiff -y main.binary maing.binary
```

![](https://cdn.jsdelivr.net/gh/Outsider565/ImageBed@master/20210329221556.png)

事实上，添加的部分为debug section，可以用readelf读出来

```shell
readelf -wi main # No output
readelf -wi maing
```

> ```
> i
> =info
> ```
>
> Displays the contents of the ‘.debug_info’ section. Note: the output from this option can also be restricted by the use of the --dwarf-depth and --dwarf-start options.

#### C.

使用ldd分析任意一个可执行文件，可看出在我的电脑上(Ubuntu20.04 aarch64)，动态链接库如下

```shell
>ldd /usr/bin/ls                
	linux-vdso.so.1 (0x0000ffff81faf000)
	libselinux.so.1 => /lib/aarch64-linux-gnu/libselinux.so.1 (0x0000ffff81f05000)
	libc.so.6 => /lib/aarch64-linux-gnu/libc.so.6 (0x0000ffff81d92000)
	/lib/ld-linux-aarch64.so.1 (0x0000ffff81f7f000)
	libpcre2-8.so.0 => /lib/aarch64-linux-gnu/libpcre2-8.so.0 (0x0000ffff81d04000)
	libdl.so.2 => /lib/aarch64-linux-gnu/libdl.so.2 (0x0000ffff81cf0000)
	libpthread.so.0 => /lib/aarch64-linux-gnu/libpthread.so.0 (0x0000ffff81cc0000)
```

- linux-vdso.so.1
- libselinux.so.1 => /lib/aarch64-linux-gnu/libselinux.so.1 
- libc.so.6 => /lib/aarch64-linux-gnu/libc.so.6
- /lib/ld-linux-aarch64.so.1
- libpcre2-8.so.0 => /lib/aarch64-linux-gnu/libpcre2-8.so.0 
- libdl.so.2 => /lib/aarch64-linux-gnu/libdl.so.2
- libpthread.so.0 => /lib/aarch64-linux-gnu/libpthread.so.0 

## 补充习题

### 1.补充源码

源码和Makefile见附带的文件

编译成功后的截图如下

<img src="https://cdn.jsdelivr.net/gh/Outsider565/ImageBed@master/20210329195524.png" style="zoom:50%;" />

### 2. 手动编译

使用了两种方法，第一种直接使用`gcc -v`的编译参数，第二种使用更简单的手动链接

#### gcc -v式的

![](https://cdn.jsdelivr.net/gh/Outsider565/ImageBed@master/20210329200556.png)

能正常编译并运行

#### 手动链接式的

在我们进行链接之前，需要先解释各个系统库有什么用，才能进行选择性的简化链接

- libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f3496d10000) - C标准库动态共享对象
- /lib64/ld-linux-x86-64.so.2 (0x00007f3497303000) - 动态链接器/加载器
- crt1.o：包含入口函数_start(该函数执行了上一节所描述的整个程序执行过程)，以及未定义的符号__libc_start_main、main
- crti.o: 提供.init节和.fini节的序言(function prologs)
- crtn.o: 提供.init节和.fini节的尾言(function epilogs)
- crtbegin.o: 提供构造函数的首地址(但在本题中，因为没有构造函数，因此不需要链接)
- crtend.o: 提供析构函数的首地址(同上)

具体到本题中，使用如下命令即可

```shell
crt1.o=/usr/lib/x86_64-linux-gnu/crt1.o
crti.o=/usr/lib/x86_64-linux-gnu/crti.o
crtn.o=/usr/lib/x86_64-linux-gnu/crtn.o
crtbeginS.o=/usr/lib/gcc/x86_64-linux-gnu/7/crtbeginS.o
crtendS.o=/usr/lib/gcc/x86_64-linux-gnu/7/crtendS.o
ld.so=/lib64/ld-linux-x86-64.so.2
libc.so=/usr/lib/x86_64-linux-gnu/libc.so
ld -dynamic-linker $(ld.so) -o main $(crt1.o) $(crti.o) main.o BubbleSort.o add.o printResult.o $(libc.so) $(crtn.o)
```

### 3.OBJDUMP进行反汇编

使用`objdump -S main.o > main.S`将反编译形成的汇编码写入`main.S`文件中

![](https://cdn.jsdelivr.net/gh/Outsider565/ImageBed@master/20210329201203.png)

### 4. 生成调试信息后，使用GDB进行调试

使用了list, break, start, run, continue, info等多种命令进行了调试，在此处限于截图篇幅，仅截取部分

![](https://cdn.jsdelivr.net/gh/Outsider565/ImageBed@master/20210329201939.png)

### 5. 相关问题

1. 分析同一个源程序在不同机器上生成的可执行目标代码是否相同

   1. ISA: 在arm处理器和x86处理器上，显然有不同的可执行目标文件代码
   2. OS：在windows和linux上有不同的寄存器分配规则，在windows64上的函数调用采用`RCX`,`RDX`,`R8`,`R9`的寄存器调用顺序，在Linux64上采用的函数调用寄存器为`RDI`,`RSI`,`RDX`,`RCX`,`R8`,`R9`
   3. 编译器：不同编译器会采用不同的优化策略，生成的可执行目标代码可能不同

2. 你能在可执行目标文件中找出函数printf ()对应的机器代码段吗？能的话，请标示出来。

   不能，printf函数所对应的机器代码段在libc.so中，是在运行时调用的，因此无法找到。

   事实上，的确有printf@plt（Procedure linkage table)这一函数，但这仅仅是一个找到真实printf代码的stub

   ```assembly
   0000000000000710 <printf@plt>:
    710:	ff 25 9a 18 20 00    	jmpq   *0x20189a(%rip)        # 201fb0 <printf@GLIBC_2.2.5>
    716:	68 03 00 00 00       	pushq  $0x3
    71b:	e9 b0 ff ff ff       	jmpq   6d0 <.plt>
   ```

3. 为什么源程序文件的内容和可执行目标文件的内容完全不同？

   源程序文件的内容为文本文件，是编程语言代码；可执行目标文件为CPU可以运行的二进制代码，自然不同。

