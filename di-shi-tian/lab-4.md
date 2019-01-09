# 抢占式多任务处理

## 简介
在同时有多个活动进程的情况下,我们将完成抢占式多任务处理.

### Part A 
1. 在JOS添加多CPU支持
2. 实现进程循环调度
3. 基本的进程管理系统调用(创建和销毁进程,分配和映射内存)

### Part B
1. 完成Unix风格的`fork()`实现,用户进程将通过它创建自身的拷贝

### Part C
1. 完成进程间通信IPC,允许用户进程互相通信和同步.
2. 添加硬件时钟中断处理

## 准备工作
1. 如果使用原生jos代码,将jos代码切换到lab4分支,并将lab3分支merge进来.
2. 如果使用我的库,则已经是完成好的代码.

### 新增代码和解释
1. kern/cpu.h
Kernel-private definitions for multiprocessor support

2. kern/mpconfig.c	
Code to read the multiprocessor configuration

3. kern/lapic.c	
Kernel code driving the local APIC unit in each processor

4. kern/mpentry.S	
Assembly-language entry code for non-boot CPUs

5. kern/spinlock.h	
Kernel-private definitions for spin locks, including the big kernel lock

6. kern/spinlock.c	
Kernel code implementing spin locks

7. kern/sched.c	
Code skeleton of the scheduler that you are about to implement

### 时间安排
Lab4共分为3个Part,每个Part一周.

















































































---