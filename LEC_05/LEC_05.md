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

# xv6系统调用的实现
xv6 进程/栈 图解
1. 用户进程,内核线程
2. 用户栈, 内核栈
3. 机制: 用户空间/内核空间切换
4. 机制: 内核线程间切换
5. 陷阱
6. 内核函数调用
7. 上下文结构体

## 简化的xv6虚拟地址空间
```
  FFFFFFFF:
            ...
  80000000: kernel
            user stack
            user data
  00000000: user instructions
```
内核通过配置MMU,用户代码仅允许访问内存地址空间的下半部分,即用户空间部分.
而每个进程映射的内核空间部分都是一样的,即将相同的物理内存地址,映射到相同的虚拟空间地址.

## 用户空间: 系统调用入口点
准备进入系统调用之前,以`write()`系统调用为例:
```
  break *0xb90
  x/3i 0xb8b
    0x10 in eax is the system call number for write
  info reg
    cs=0x1b, B=1011 -- CPL=3 => user mode
    esp and eip are low addresses -- user virtual addresses
  x/4x $esp
    cc1 is return address -- in printf
    2 is fd
    0x3f7a is buffer on the stack
    1 is count
    i.e. write(2, 0x3f7a, 1)
  x/c 0x3f7a
```

## INT指令: 内核入口
```
stepi
info reg
    cs=0x8 -- CPL=0 => kernel mode
    note INT changed eip and esp to high kernel addresses
  where is eip?
    at a kernel-supplied vector -- only place user can go
    so user program can't jump to random places in kernel with CPL=0
  x/6wx $esp
    INT saved a few user registers
    err, eip, cs, eflags, esp, ss
```

1. 为什么INT指令保存了上面的哪些寄存器?
因为在中断处理的过程中,内核可能覆盖这些寄存器.

2. INT指令做了哪些事情?
* 切换到当前进程的内核栈
* 在内核栈中保存必要的用户寄存器
* 将CPL设置为0
* 从内核提供的中断向量入口点`vector`开始执行代码.

3. 内核%esp从何而来?
内核栈在进程创建的时候被分配.通过%esp寄存器,内核告知硬件当前进程的内核栈顶位置.

Q: 为什么INT指令有且只保存了这些指令?

## 用户空间信息的保存
通过INT指令,保存了一部分用户空间信息.

通过`trapasm.S`中的`alltraps`,将会保存用户空间剩下的寄存器.通过`pushal`将会一次性将8个寄存器入栈:`eax~edi`.

```
x/19x $esp
  19 words at top of kernel stack:
    ss
    esp
    eflags
    cs
    eip
    err    -- INT saved from here up
    trapno
    ds
    es
    fs
    gs
    eax..edi
```

阅读`x86.h`中的`trapframe`结构体,有助于这部分的了解.本质上就是在系统调用后,整个用户空间的信息都将保存在`trapframe`中.当从系统调用返回后,将通过`trapframe`恢复用户空间状态.有时在内核执行过程中,将会读写`trapframe`的内容.

Q: 用户空间信息为何要保存在内核栈,而不是用户栈?


## 系统调用处理流程

### 进入内核C代码
目前来看在INT指令之后,将会跳转到Vectors.S执行.在将trapno入栈后,将会跳转到`alltraps`.
`alltraps`将用户空间剩余信息入栈后,将栈顶指针%esp的值入栈,此时%esp也是`trapframe`的地址,然后跳转到`trap`函数.

### 内核 系统调用处理
1. 设备中断和系统异常也会进入`trap`函数.
2. 系统调用的`trapno`为`T_SYSCALL`.
3. `myproc()`将会返回当前核运行的进程结构体`proc`.
4. `proc`定义在`proc.h struct proc`.
5. 进入`trap`后,首先将`trapframe`保存到当前进程的`proc`结构体中.
6. `syscall()`可以从先前保存的`trapframe %eax`中,获取系统调用号.
7. 通过系统调用号,将在`syscalls`表中执行相应的函数,并将结果保存回`trapframe->eax`.
8. 以系统调用`write()`为例,对应系统调用号为`0x10`,在`syscalls`表中对应函数为`sys_write`.
9. 通过`arg*()`系列函数,从用户空间栈上,将参数读取到内核栈中.
10. 最后调用底层函数,完成功能.

### 恢复用户空间
1. `trapframe->eax`保存着系统调用的返回值.
2. `syscall()`之后,会返回`trap()`,接着返回`trapasm.S`.
3. 之后即将进入`trapret`,是之前调用系统调用的逆过程.即将内核栈出栈.
4. `iret`指令是`INT`指令的逆指令:
  * 恢复寄存器eip, cs, eflags, esp, ss
  * 设置CPL为3
5. 至此,我们已经恢复到用户空间.

Q: 我们真的需要`iret`指令么?
1. 可以用普通指令代替iret指令么?
2. 可以简化`iret`指令么?

## fork
下面我们来考察`fork()`是如何创建一个子进程的,尤其是子进程是如何返回用户空间的.

主要逻辑如下:
1. 前面我们已经知道系统调用是如何返回的,具体来说就是`trapret`会根据之前保存的`trapframe`来恢复用户空间.
2. 类似地,`fork()`将会在`trap()`中,构造一个假的`trapframe`.
3. 子进程将在内核中被调度执行.
4. 子进程好像是通过系统调用下来一样,被处理返回.

注意这里有两个独立的动作:
1. 创建一个新的进程.
2. 执行新的进程.

## fork代码分析
`fork()`的代码执行主要分为两个部分:

1. 调用`allocproc()`
  * 分配了一个新的`proc`结构体.
  * 为`p->kstack`分配了内存
  * 为`trapframe`预留了空间
  * 设置子进程从内核返回用户空间的函数地址为`trapret`
  * 设置子进程内核空间%eip为`forkret`
  
2. 再次返回`fork()`,此时主要目的就是填充新的`proc`结构体.
  * 为子进程分配并映射内存页
  * 将当前进程用户空间页表复制给子进程
  * 将当前进程的trapframe复制给子进程
  * 设置子进程的返回值,即`trapframe->eax`为0.
  * 将当前进程的打开文件数组复制给子进程
  ...
  * 最终当前进程返回值为子进程的进程号
  
### 子进程的内核栈
```
  trapframe -- copy of parent, but eax=0
  trapret's address
  context
    eip = forkret
```
















----