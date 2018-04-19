# 隔离机制
1. 用户空间/内核空间 隔离
2. xv6系统调用

## 如何概括内核的作用?
1. 设备驱动库,用于联系硬件和应用程序.
2. 用于在硬件上运行应用程序.
3. 更高效灵活地使用硬件,通常支持多任务.

## 多任务的必要条件
1. 复用硬件.
2. 多个任务间互相隔离.
3. 多个任务间互相通信.

## 多任务的设计方法
1. 核心: 对资源进行抽象,而非使用原始资源,从而实现对硬件的共享.
2. 使用文件系统,而非直接操作原始硬盘.
3. 使用进程的概念来进行资源管理,而非直接使用CPU和物理内存.
4. 网络通信使用socket来进行抽象,而非原始以太网数据包.

正是通过对硬件资源的抽象才实现了资源的共享.如此一来,应用程序不再需要关心硬件资源,从而实现了更好的可移植性.

## 什么是多任务间隔离?
1. 一个进程通常就是一个隔离的单元.
2. 进程中的错误和失败必须被隔离,不能影响其他进程和内核.
3. 进程X必须无法篡改读取进程Y的内存,文件描述符.
4. 进程X必须无法永久占用CPU.
5. 进程必须无法篡改攻击内核.

## 多任务间隔离的硬件机制
内核使用了硬件提供的隔离机制
1. 用户模式/内核模式的标志位
2. 地址空间
3. 时间分片
4. 系统调用接口

## 硬件: 用户模式/内核模式标志位
1. 用于控制指令是否可以操作特权硬件.
2. 在x86上标志位被称为CPL.它使用了%CS寄存器的最低的两个比特位.
3. CPL=0,被称为内核模式,具有特权级.CPL=3,被称为用户模式,非特权级.
4. CPL保护了很多与隔离机制相关的资源
    
    4.1 IO端口访问权限 
    4.2 控制寄存器访问权限: eflags, %CS, %CS4
    4.3 间接影响了内核访问权限
5. 几乎每个CPU都有类似的用户/内核模式标志位机制.

## 如何通过切换CPL标志位来设计系统调用
* 用户空间先切换标志位,再跳转到系统调用?
```
set CPL=0
jmp sys_open
```
不行,用户空间不应该具有切换标志位的能力.

* 通过类似CALL的混合指令,用户空间在跳转到系统调用的同时切换标志位?
不行,用户空间可能是恶意的.因此可能跳转到内核的任意地址.

* x86的做法:
1. 内核只提供了非常有限的入口,被称为中断向量.
2. 指令INT将CPL设置为0,同时跳转到相应的内核入口.
3. 用户空间不能直接修改CPL,也无法跳转到内核任意地址.
4. 内核空间在返回用户空间前,将CPL置为3.这同样是一个混合指令,即返回用户空间和CPL置3是无法分割的.

## 用户模式调用内核的约定
1. CPL=3,用户模式,非特权级,执行用户空间指令.
2. CPL=0,内核模式,特权级,执行内核入口处的指令,即中断向量.

以下情况应该避免:
1. CPL=0,内核模式,执行用户空间指令.
2. CPL=0,内核模式,执行用户任意指定的内核指令.

## 如何隔离进程的内存?
1. 方法: 使用*地址空间*的概念
2. 为一个进程分配一定的内存空间,用于存放指令,变量,堆,栈.
3. 阻止进程访问未被分配或未授权的内存空间,比如内核的内存,其他进程的内存.

## 如何创建独立的内存地址空间?
1. xv6使用了x86内存管理单元(MMU)提供的硬件分页机制.
2. MMU为进程的地址提供了一种映射机制.进程使用的都是虚拟地址,通过MMU的映射,转换为物理地址.
```
    CPU -> MMU -> RAM
            |
         pagetable
    VA ---------> PA
```
3. 一旦启动分页机制之后,MMU会为所有内存地址提供映射.用户空间的指令和数据,内核空间的指令和数据都必须经过MMU的转换.
4. 指令只会使用虚拟地址,不会直接使用物理地址.
5. 内核为每个进程建立了一个不同的页表(page table),每个进程的页表规定了该进程只能访问自己的内存地址空间.

## xv6系统调用的实现
















----