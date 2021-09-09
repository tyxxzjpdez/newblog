---
title: gdb常用调试指令
date: 2021-09-09 14:43:01
tags:
  - debug
---

## 背景		

最近在看《Linux环境编程：从应用到内核》，涉及到了比较多的内核代码，恰好之前尝试过调试内核，但那时由于对内核无从下手，很容易一进门就迷失，进而就失去兴趣了。这次打算以这本书作为一个线索，不过多地追究细节，快速定位并搞懂关键性概念。

如何创建调试环境虽然比较麻烦，但网上教程较多，这里不多赘述，具体可参看以下三篇博客：

* [使用 linux kernel +busybox 定制linux系统](https://www.cnblogs.com/CtripDBA/p/12290071.html)

* [使用QEMU和GDB调试Linux内核](https://consen.github.io/2018/01/17/debug-linux-kernel-with-qemu-and-gdb/)

* [搭建Linux kernel调试环境-busybox构建最小根文件系统](https://blog.csdn.net/zhaojia92/article/details/100717580)

本文重点是搭建调试环境以后如何使用gdb进行调试

## gdb配置

虽然现在随着gdb版本的提高，功能也越来越完善，但是经过配置完全可以变得更友好得多，这里主要使用[pwndbg](https://github.com/pwndbg/pwndbg) 这个工具。当然，在内核调试该工具相应比较慢，甚至出现半天不响应的情况，所以可能存在bug，但是在一般的项目中是足够的，曾经就使用它查看过动态库的载入，默认配置就已经很舒服，很直观。

## gdb命令

### 建议的额外编译选项

编译建议加上`-ggdb`，从`man gdb`可以看到这对于gdb可以产生几乎最多的调试信息

### 开始调试

* `gdb program` 一般指从没有运行的程序开始调试
* `gdb program core` 指定可执行文件和core文件
* `gdb program 123`或者`gdb -p 1234`指定调试一个已经运行的程序，此时一般需要调试的程序与被调试的程序之间是祖先与子孙关系，否则就需要root权限

### 常用命令

* **help cmd**         #查看相关命令的使用方法
* **start**                  #开始调试，并停在第一行代码
* **list**                     #查看当前位置上下的源代码
* **breakpoints  \<lines\>/\<func\>**        #在制定行或者函数设置断点
* **info breakpoints**                             #查看当前设置的断点
* **delete breakpoints [NO]**               #删除指定编号的某个断点，或删除所有断点。断点编号从1开始递增
* **step**                  #执行一行源程序代码，如果此行代码中有函数调用，则进入该函数
* **next**                  #执行一行源程序代码，此行代码中的函数调用也一并直接执行，不跳入
* **run**                   #运行被调试的程序。如果此前没有下过断点，则执行完整个程序；如果有断点，则程序暂停在第一个可用断点处
* **continue**         #继续执行被调试程序，直至下一个断点或程序结束
* **finish**               #函数结束
* **backtrace**       #查看当前栈帧
* **layout asm/next**                               #使用asm或者next布局，比较友好，详情看help
* **print/display/disassemble 未完待续**

## 参考

* [GDB调试基本命令](https://blog.csdn.net/mercy_ps/article/details/81542986)
* `man gdb`
