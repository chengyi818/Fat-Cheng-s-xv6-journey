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

`kern/init.c`中的函数`boot_aps()`负责完成APs的启动过程.APs会以实模式启动,非常类似于BSP执行`boot/boot.S`的过程,所以`boot_aps()`首先将APs的启动代码拷贝到实模式下的地址.和bootloader启动不同的是,我们可以控制APs开始执行代码的位置.JOS将入口代码拷贝到`0x7000`(MPENTRY_CODE)开始的物理地址,但是低于640KB,任何未使用的,页对齐的物理地址其实都可以.

此后,`boot_aps()`函数将通过发送STARTUP IPIs和初始的CS:IP到相应AP的LAPIC单元,一个接一个的激活AP.AP应该在CS:IP指定的地址开始运行其入口代码(在JOS中是MPENTRY_PADDR 0x7000).AP的入口代码`kern/mpentry.S`和`boot/boot.S`非常相似.经过短暂的设置后,AP将开启保护模式并启用分页.然后将调用C程序`kern/init.c`中的`mp_main()`函数.`boot_aps()`等待AP在其对应的CpuInfo中将`cpu_status`置为CPU_STARTED标志,然后再唤醒下一个CPU.


### Exercise 2

阅读`kern/init.c`中的`boot_aps()`和`mp_main()`函数,以及`kern/mpentry.S`.确保理解了APs的启动流程.然后修改`kern/pmap.c`中`page_init()`的实现,以避免将`MPENTRY_PADDR`所在的物理页放入`page_free_list`,这样我们才能安全地拷贝和运行AP的启动代码.

修改完成后,代码应当可以通过`check_page_free_list()`的检查.(可能无法通过`check_kern_pgdir()`的检查,稍后我们会修复这个问题.)

### Q&A
对比`kern/mpentry.S`和`boot/boot.S`,`mpentry.S`被编译和链接到`KERNBASE`之上运行,宏`MPBOOTPHYS`的作用是什么?为什么`kern/mpentry.S`需要,而`boot/boot.S`不需要?换言之,如果`mpentry.S`去掉宏`MPBOOTPHYS`会发生什么错误?

提示:
回想一下我们在Lab 1中讨论过的关于链接地址和加载地址的区别.
[stack overflow](https://stackoverflow.com/questions/33859072/what-does-pc-have-to-do-with-load-or-link-address)

答:
加载地址是实际存储指令的位置,而链接地址是链接器为指令中变量生成地址的基地址.

对于`boot/boot.S`而言,链接地址为0,加载地址也为0,因此并不受影响.
`kern/mpentry.S`,链接地址为0,加载地址为`MPENTRY_CODE`,而其中的变量地址并没有被重定位.此时如果使用linker填充的地址将无法正常访问到变量.必须通过宏`MPBOOTPHYS`来进行重定位转换.


---

## 各CPU状态和初始化
 






















































----