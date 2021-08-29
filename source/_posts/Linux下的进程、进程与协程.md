---
title: 结合源码从开销角度看Linux下的进程、进程与协程
date: 2021-08-29 20:04:28
tags:
  - Linux
  - process
  - thread
  - coroutine
---

## 问题

最近面试的时候被问了两个问题：

* 进程、线程、协程的关系与区别
* 为什么要使用线程池，相比普通的方式有什么优势？

两个问题其实不算很难回答，我主要从开销角度讲的。但是面试官随后便问了我到底是什么开销呢？我一下就懵了。乖乖，这不看过源码谁知道，所以就有了这篇博客。

## 分析

### 进程与线程

**本文的内容主要基于《用“芯”探核：基于龙芯的Linux内核探索解析》以及一点点《CSAPP》的内容。**

在Linux下新建一个进程或者线程都会调用`fork`族的系统调用，也就是`fork`,`vfork`,`clone`。他们最终都是在`_do_fork(struct kernel_clone_args*)`函数内最终具体实现 。该函数的参数比较复杂，其具体指定了不同的创建方式。这里仅仅列举以下几个：

| 标志名称        | 标志含义                                 |
| --------------- | ---------------------------------------- |
| `CLONE_VM`      | 子进程共享父进程的**地址空间和各级页表** |
| `CLONE_FS`      | 子进程共享父进程的文件系统上下文         |
| `CLONE_FILES`   | 子进程共享父进程的打开的文件描述符表     |
| `CLONE_SIGHAND` | 子进程共享父进程的信号处理函数表         |
| `CLONE_VFORK`   | 子进程执行新程序或退出之前父进程阻塞     |
| `CLONE_THREAD`  | 将子进程**插入父进程所在的线程组**       |
| `CLONE_SETTLS`  | 子进程将创建自己的线程本地存储           |

而这三个系统调用大概是使用了其中几个标志

* `fork`函数没有特殊标志
* ` vfork`则有`CLONE_VFORK`和`CLONE_VM`标志
* `clone`一般用作创建线程。我们知道一般情况下`std::thread`创建线程的方式是依赖于pthread动态链接库的（ 源码上赫然写着`pthread_create`，而且使用命令行编译链接，不加上-lpthread也会链接错）。而它的底层实现使用了如下标志`CLONE_VM`、`CLONE_FS`、`CLONE_FILES`、`CLONE_SIGNAL`、`CLONE_SETTLS`、`CLONE_PARENT_SETTID`、`CLONE_CHILD_CLEARTID`、`CLONE_SYSVSEM`。所以Pthread库创建出来的线程**共享父进程的地址空间、文件系统上下文、文件描述符、信号及其处理函数、SysVIPC信号量取消队列**，和父进程处于同一线程组，创建自己的线程本地存储，将子进程的PID返回给调用者的`parent_tidptr`字段，并在执行新程序或退出时清空`child_tidptr`字段同时唤醒等待该事件的进程。

其实从这里我们便可见一斑了。线程比进程的创建多了很多标志，而且这些标志大多是声明自己与父进程共享一些资源。也就是说从开销来说，相比一般的进程，线程就是共享代替创建，所以就少了很多新建与维护代价。我们以`CLONE_VM`这个标志为例简单地过一遍代码（内核版本5.2）：

首先是`_do_fork`

```cpp
long _do_fork(unsigned long clone_flags,
	      unsigned long stack_start,
	      unsigned long stack_size,
	      int __user *parent_tidptr,
	      int __user *child_tidptr,
	      unsigned long tls)
{
    //...
p = copy_process(clone_flags, stack_start, stack_size, parent_tidptr,
			 child_tidptr, NULL, trace, tls, NUMA_NO_NODE);
	//...
	pid = get_task_pid(p, PIDTYPE_PID);
	nr = pid_vnr(pid);
//...
	return nr;
}
```

copy_process负责对进程数据结构`task_struct`根据标志进行拷贝。

```cpp
static __latent_entropy struct task_struct *copy_process(
					unsigned long clone_flags,
					unsigned long stack_start,
					unsigned long stack_size,
					int __user *parent_tidptr,
					int __user *child_tidptr,
					struct pid *pid,
					int trace,
					unsigned long tls,
					int node)
{
    //...
    // 拷贝现有task_struct，返回局部变量p
    p = dup_task_struct(current, node);
    //...
    retval = copy_semundo(clone_flags, p);
	if (retval)
		goto bad_fork_cleanup_security;
	retval = copy_files(clone_flags, p);
	if (retval)
		goto bad_fork_cleanup_semundo;
	retval = copy_fs(clone_flags, p);
	if (retval)
		goto bad_fork_cleanup_files;
	retval = copy_sighand(clone_flags, p);
	if (retval)
		goto bad_fork_cleanup_fs;
	retval = copy_signal(clone_flags, p);
	if (retval)
		goto bad_fork_cleanup_sighand;
	retval = copy_mm(clone_flags, p);
	if (retval)
		goto bad_fork_cleanup_signal;
	retval = copy_namespaces(clone_flags, p);
	if (retval)
		goto bad_fork_cleanup_mm;
	retval = copy_io(clone_flags, p);
	if (retval)
		goto bad_fork_cleanup_namespaces;
	retval = copy_thread_tls(clone_flags, stack_start, stack_size, p, tls);
	if (retval)
		goto bad_fork_cleanup_io;
    //..
```

接下来进入`copy_mm`，看标志位究竟如何影响拷贝

```cpp
static int copy_mm(unsigned long clone_flags, struct task_struct *tsk)
{
    //...
    oldmm = current->mm;
	if (!oldmm)
		return 0;
	//...
	if (clone_flags & CLONE_VM) {
		mmget(oldmm);
		mm = oldmm;
		goto good_mm;
	}
	retval = -ENOMEM;
	mm = dup_mm(tsk, current->mm);
	if (!mm)
		goto fail_nomem;
good_mm:
	tsk->mm = mm;
	tsk->active_mm = mm;
	return 0;
fail_nomem:
	return retval;
```

 这里就已经很能说明问题了，如果有了`CLONE_VM`标志就直接指向相同的指针，代价仅为一次赋值而已，但是如果没有这个标志的话，就要调用`dup_mm`了，即使在COW（写时复制）的情况下，可以不复制内存页面mmap，但是至少也必须得复制页表，否则压根没有办法实现写时复制了嘛。

### 线程与协程

有急事。待续。

### 总结
