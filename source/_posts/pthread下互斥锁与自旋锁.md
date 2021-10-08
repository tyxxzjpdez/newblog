---
title: pthread下互斥锁与自旋锁
date: 2021-10-08 19:09:22
tags:
  - linux
  - multi-thread
---

本文主要从源码角度探讨工业级别的互斥锁与自旋锁的具体设计。

## 互斥锁pthread_mutex_lock

此处为了便于理解，仅仅展示伪代码（下图来自《Linux环境编程：从应用到内核》P307）

<img src="/home/tyxxzjpdez/图片/2021-10-08 19-28-53屏幕截图.png" alt="2021-10-08 19-28-53屏幕截图"  />

### 核心思想

* 维护一个原子量，多线程共享该值，且该值初始化为0
  * 该值为0表示互斥量没有上锁，进入之后立即返回即可
  * 该值为1表示现在互斥量已经上锁，但是没有线程正在等待
  * 该值为2表示互斥量已经上锁，并且有线程在等待该锁
* 当发现互斥量为1时不要直接睡眠，而是在while循环中自旋地询问该值，只要该值为0，便立马继续下去
* 只有当互斥量为2时才会直接考虑睡眠

### 实现优点

该方法最大的优点是当争抢不严重的时候，并不需要陷入内核，退化为了一个自旋锁，提高了效率。

## 自旋锁pthread_spin_lock

这里的源码就很简单了。

### 环境：

* ubuntu18.04LTS x86_64

* gcc 版本11.1.0
* glibc 版本 2.17

### pthread_spin_lock

```asm
   0x0000000000404d80 <+0>:	lock decl (%rdi)
   0x0000000000404d83 <+3>:	jne    0x404d90 <pthread_spin_lock+16>
   0x0000000000404d85 <+5>:	xor    %eax,%eax
   0x0000000000404d87 <+7>:	ret    
   0x0000000000404d88 <+8>:	nopl   0x0(%rax,%rax,1)
   0x0000000000404d90 <+16>:	pause  
   0x0000000000404d92 <+18>:	cmpl   $0x0,(%rdi)
   0x0000000000404d95 <+21>:	jg     0x404d80 <pthread_spin_lock>
   0x0000000000404d97 <+23>:	jmp    0x404d90 <pthread_spin_lock+16>

```

### pthread_spin_unlock

```asm
   0x0000000000404da0 <+0>:	movl   $0x1,(%rdi)
   0x0000000000404da6 <+6>:	xor    %eax,%eax
   0x0000000000404da8 <+8>:	ret 
```

### 分析

逻辑上其实很简单，自旋加锁循环查看原子量是否为0，而解锁直接将该原子量递增即可。比较有意思的是加锁部分

* lock是RMW操作，逻辑上可以认为锁住该cacheline的总线，原子修改该内存值，并且具有双向内存屏障的作用。
* 在spin_lock函数中为什么需要pause指令，它有什么功能？[^1][^2]
  * 首先需要明确虽然x86拥有强的内存模型，仅允许很少的乱序重排。但是为了提高性能，在具体实现上，CPU实际上依旧可能是有先后顺序的，但是在执行之后，CPU会做一个检测，如果满足则继续，但是如果不满足，就会产生一个`memory ordering violation`，就需要重新改变流水线再重新执行，这就大大降低了CPU执行的性能。
  * 在spin_lock中可能需要持续读取其他cpu上的共享变量，这时为了保证本地CPU察觉到这种改变，可能就经常会发生`memory ordering violation`，所以经常打断原本的流水线，大大降低CPU执行速度。
  * 另外一般spin_lock并不需要过高的时间粒度，过高的CPU占用率可能反而是只是浪费能源。
  * 综合以上两点，pause的基本功能是让CPU休眠几个时钟周期后再被唤醒，不仅不会过于频繁地发生读取从而导致`memory ordering violation`，而且休眠也能降低功耗。
* 为什么spin_lock需要加上LOCK前缀，而spin_unlock却不需要？[^3]
  * 使用spin_unlock的时候已经离开临界区了，而且本身是一个函数，开头的第一个指令一般不可能与前面的指令发生重排，而且考虑到`movl   $0x1,(%rdi)`是一个store操作，我们根本不关注它本身是否会推迟，那根本无所谓，因为无论如何都依旧符合store release的语义。
  * 而使用spin_lock的时候才刚开始临界区，需要满足load acquire的语义，不希望该原子量后面的指令跑到前面去，所以需要使用LOCK指令。

## 参考

[^1]: [自旋锁spinlock剖析与改进](https://kb.cnblogs.com/page/105657/)

[^2]: [Why flush the pipeline for Memory Order Violation caused by other logical processors?](https://stackoverflow.com/questions/55563077/why-flush-the-pipeline-for-memory-order-violation-caused-by-other-logical-proces)
[^3]: [How come spin-unlock does not need to flush store buffer on x86/amd64 ](https://stackoverflow.com/questions/68090562/how-come-spin-unlock-does-not-need-to-flush-store-buffer-on-x86-amd64)
