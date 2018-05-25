# Lab2 内存管理

## 介绍
本节试验中,我们将会完成内存管理单元.内存管理分为两部分.

1. 内核物理内存分配单元.内核将通过它来分配,释放内存.内存的管理单位为4k,称为一个页.我们的任务包括记录已分配,未分配的物理页.处理多进程共享物理页.
2. 虚拟内存单元.虚拟内存单元实现了虚拟内存和物理内存的映射关系.x86的内存管理单元(MMU)中保存了这样的映射关系(page table).

## 准备工作
首先将代码分支切换到lab2.这里我为了将自己的代码保存到github上.我使用了两个上游repo,一个是mit的repo用于更新代码.另一个是github的repo,用于保存代码.

相关命令请参考github[Configuring a remote for a fork](https://help.github.com/articles/configuring-a-remote-for-a-fork/)

## lab2 新增文件
```
inc/memlayout.h
kern/pmap.c
kern/pmap.h
kern/kclock.h
kern/kclock.c
```

1. memlayout.h描述了jos的虚拟地址布局,我们需要通过修改pmap.c来实现这样的布局.
2. 在memlayout.h中我们定义了`PageInfo`结构体.通过该结构体,我们一方面管理剩余可用物理内存,另一方面管理物理内存页的引用计数.`PageInfo`和物理页之间存在一一对应关系.
3. `kclock.c`和`kclock.h`用于操作Pc时钟和CMOS RAM.在CMOS RAM中,BIOS保存了PC的物理内存大小等信息.
4. `pmap.c`中的代码需要读取CMOS RAM,以知晓当前可用的物理内存大小.在jos中,这部分代码已经完成了.
5. 请注意阅读`memlayout.h`,`pmap.h`和`mmu.h`.我们需要理解其中的定义,才能顺利完成课程.