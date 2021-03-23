---
title: ELF头简介
date: 2021-03-23 11:48:36
index_img: /img/ICS/binary.gif
category: [ICS]
tags: [Linker]
---

​	在linux中我们常用readelf指令来读取ELF (Executable and Linkable Format) 文件中的信息，本文主要介绍ELF头的基本信息

![ELF文件结构](https://upload.wikimedia.org/wikipedia/commons/thumb/7/77/Elf-layout--en.svg/260px-Elf-layout--en.svg.png)

### ELF头

elf头是位于elf文件的头部，里面存储着一些机器和该ELF文件的基本信息。

```c
typedef struct {
        unsigned char   e_ident[EI_NIDENT];
        Elf64_Half      e_type;
        Elf64_Half      e_machine;
        Elf64_Word      e_version;
        Elf64_Addr      e_entry;
        Elf64_Off       e_phoff;
        Elf64_Off       e_shoff;
        Elf64_Word      e_flags;
        Elf64_Half      e_ehsize;
        Elf64_Half      e_phentsize;
        Elf64_Half      e_phnum;
        Elf64_Half      e_shentsize;
        Elf64_Half      e_shnum;
        Elf64_Half      e_shstrndx;
} Elf64_Ehdr;
```

我们分别介绍其含义

---

#### 1、e_ident

- **长度：16字节**
- **简介：包含着文件和操作系统信息**
- <img src="https://tva1.sinaimg.cn/large/008eGmZEly1gosklimwgzj30m00fmq55.jpg" style="zoom:50%;" />

##### Magic Num - e_ident[0:3]

​	前四个字节包含着一个 magic number，表示该文件是一个 ELF 文件

##### EI_Class - e_ident[4]

​	指示文件类型，是ELF32还是ELF64位

<img src="https://tva1.sinaimg.cn/large/008eGmZEly1goskl3oqhyj30dq05qq3h.jpg" style="zoom:50%;" />

##### EI_DATA - e_ident[5]

​	指示文件的编码方式，是大端法还是小端法

<img src="https://tva1.sinaimg.cn/large/008eGmZEly1goskkp5pdjj30fy05ggm5.jpg" style="zoom:50%;" />

​	**ELFDATA2LSB - 小端法**

​	**ELFDATA2MSB - 大端法**

##### EI_Version - e_ident[6]

​	标识ELF Version, 该值等于EV_CURRENT，目前为1

##### EI_OSABI - e_ident[7]

​	表示着该文件运行的操作系统

<img src="https://tva1.sinaimg.cn/large/008eGmZEly1goskk8xx2wj30oi0jmadb.jpg" alt="操作系统类型对应" style="zoom:50%;" />

##### EI_ABIVERSION - e_ident[8]

​	标志着 ABI （应用二进制接口）的版本，ABI相当于硬件层级的API（见下图）

<img src="https://tva1.sinaimg.cn/large/008eGmZEly1gosknnji29j31400u0qd8.jpg" alt="ABI解释" style="zoom:40%;" />

##### EI_PAD - e_ident[8:15]

​	填充位，用零填充用以对齐，可以预留给未来使用



#### 2、e_type

- **长度：2字节**

- **简介：**指示文件类型

  <img src="https://tva1.sinaimg.cn/large/008eGmZEly1gosm36oov3j30he0da40a.jpg" style="zoom:50%;" />

​	

#### 3、e_machine

- **长度：2字节**

- **简介：**指示机器类型

  <img src="https://tva1.sinaimg.cn/large/008eGmZEly1gosm42l3lxj30u00x2wkc.jpg" alt="部分机器类型" style="zoom:50%;" />



#### 4、e_version

​	**长度：4字节**

​	**简介：指示文件版本**

<img src="https://tva1.sinaimg.cn/large/008eGmZEly1gosm8c0knij30dk04gglx.jpg" style="zoom:50%;" />

#### 5、e_entry

​	**长度：4字节（32位）/8字节（64位）**

​	**简介：进程开始的虚拟地址**

#### 6、e_phoff

​	**长度：4字节（32位）/8字节（64位）**

​	**简介：指向程序头部表的开始**	

#### 7、e_shoff

​	**长度：4字节（32位）/8字节（64位）**

​	**简介：指向节头部表的开始**	

#### 8、e_flags

​	**长度：4字节**

​	**简介：意义取决于目标架构**	

#### 9、e_ehsize

​	**长度：2字节**	

​	**简介：该文件头部的大小**

#### 10、e_phentsize

​	**长度：2字节**	

**简介：程序头部的大小**	

#### 11、e_phnum

​	**长度：2字节**	

​	**简介：程序头部的条目数**

#### 12、e_shentsize

​	**长度：2字节**	

​	**简介：节头部表的大小**

#### 13、e_shnum

​	**长度：2字节**	

​	**简介：节头部表的条目数**

#### 14、e_shstrndx

​	**长度：2字节**	

​	**简介：节头部表的条目和其位置 (idx) 的对应关系**

---

### Reference

[1] https://en.wikipedia.org/wiki/Executable_and_Linkable_Format

[2] https://refspecs.linuxfoundation.org/elf/gabi4+/ch4.eheader.html

