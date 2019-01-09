# Part A

## 多处理器支持
首先,我们将使JOS支持**对称多处理SMP**,这是一种多处理器模型,其中所有CPU对系统资源(如内存和I/O总线)具有同等的访问权限.尽管SMP中所有CPU功能相同,但在启动引导过程中,它们可以分为两种类型:
1. 引导启动处理器(bootstrap processor,BSP),负责初始化硬件和移动操作系统.
2. 应用处理器(application processors, APs),只有在操作系统启动并运行后,BSP才会激活应用处理器APs.

至于在SMP的诸多处理器中,选择哪个称为BSP,则取决于硬件和BIOS的设置.截止到目前为止,我们所有的JOS代码均是运行在BSP上的.

在SMP系统中,每个CPU都有一个对应的本地APIC(Local Advanced Programmable Interrupt Controller, LAPIC).LAPIC负责在系统中传递中断,同时也会为它连接的CPU提供唯一标识符.在Lab 4中,我们将会使用LAPIC(kern/lapic.c)的如下基础功能:
* cpunum()
> 读取LAPIC标识符(APIC ID),以获取当前代码运行在哪个CPU上.

* lapic_startap()
> 从BSP发送STARTUP处理器间中断(interprocessor interrupt, IPI)给APs,以启动其他CPU.

* apic_init()
> Part C中,我们对LAPIC的内置计时器进行编程,以触发时钟中断,支持抢占式多任务处理.

CPU使用内存映射IO(memory-mapped I/O,MMIO)访问LAPIC.在MMIO中,物理内存会和IO设备的寄存器直接相连,因此用于访问内存的load/store指令同样可以用于访问设备寄存器.我们在前面的章节中,已经看到过MMIO的应用,比如物理地址`0xA0000`就是用于VGA显示.

在一个4GB物理内存的系统上,LAPIC映射的物理内存地址为`0xFE000000`,即4GB地址以下32MB的空间.很明显,我们无法使用从`KERNBASE`开始的直接映射来访问这么高的物理地址.JOS虚拟内存映射从MMIOBASE开始映射了一块4MB的虚拟内存空间,用于映射IO设备.后面我们会介绍更多关于MMIO的内容,现在我们的目标是写一个简单的函数来分配该区域的物理内存,并将设备内存映射到这里.

### Exercise 1
完成`kern/pmap.c`中的`mmio_map_region()`函数.我们可以在`kern/lapic.c`中的`lapic_init()`函数中,看到`mmio_map_region()`是如何被使用的.目前我们还不能单独测试`mmio_map_region()`,我们需要接着完成下一个Exercise.

---

## 启动应用处理器APs
在启动APs之前,BSP首先需要搜集当前这个多处理器系统的信息,比如CPU的总数,APIC标识符,LAPIC单元的MMIO地址.`kern/mpconfig.c`中的`mp_init()`函数通过读取BIOS内存区域中的MP配置表来获取这些信息.