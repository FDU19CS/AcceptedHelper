---
author: 曹丝露
email: 56127817+Outsider565@users.noreply.github.com
title: 0324ICSHW问题汇总
date: 2021-03-30 00:15:11
index_img: https://ss1.bdstatic.com/70cFvXSh_Q1YnxGkpoWK1HF6hhy/it/u=1872186348,775683073&fm=26&gp=0.jpg
category: [ICS]
tags: [曹丝露]
---
## 1.关于作业7.13B，为什么两个文件objdump的结果一样，但是通过diff比较二进制文件得出的结果不同？

-g主要是加了debug section，它包含了符号表，如果没有-g的内容，用gdb进行debug时，它并不知道每一行是什么内容，也不能print i变量，因为local variable会丢失，它只知道有个内存地址，不知道变量是i，但是如果使用-g选项，就可以添加这些信息。

![image-20210329234104958](https://cdn.jsdelivr.net/gh/Outsider565/ImageBed@master/20210330001135.jpg)

`objdump`相当于只读了.text内容，其他地方都没有读。

## 2.关于补充作业题中的链接时出现`segmentation fault`

### 解决方法：

![image-20210329234104958](https://cdn.jsdelivr.net/gh/Outsider565/ImageBed@master/20210330001208.jpg)

### 出错原因（此处为tyf的解释，可能复述的有不到位的地方）：

-e main是进入到main，但是此前没有执行虚拟内存映射，导致地址出错，出现segmentation fault，进入_start可以，因为 _start中会调用libc_start，（因为源码不公开，所以并不知道这其中做了什么，但是猜想它进行了虚拟内存映射），如果删去-e _start，即为默认值，也是可以正常运行的。

libc_start是定义在crt1.o中的，只要把它放在main之前，就可以进入。

### 用main也可以正常运行的方法：

不要链接crt1.o，将entry point设为main，这样会进入main，main会return，return之后立刻会有segmentation fault，此时将return 0 改为 exit 0就可以了，因为exit调用的是system call，会执行一个中断。

## 3.程序加载的过程

在进入main函数之前，首先会有一个程序准备全局变量等，最后来调用main函数，把他的参数传入，执行main函数，会return 0 给一个结束的程序，它会将一些“善后”的工作做完。如果入口是main函数，则上述准备操作均没有做，就return到一个系统不知道在哪儿的位置，所以会segmentation fault，但如果用exit就没有问题。

## 4.标准的链接格式

`ld -o OUTPUT crt1.o crti.o crtbegin.o [-L paths] [user objects] [gcc libs] [C libs] [gcc libs] crtend.o crtn.o`



感谢谭一凡的耐心解答！