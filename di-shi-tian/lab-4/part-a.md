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

CPU使用内存映射IO(memory-mapped I/O,MMIO)访问LAPIC.