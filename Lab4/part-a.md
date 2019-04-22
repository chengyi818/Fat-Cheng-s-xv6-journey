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

编写多处理器操作系统时,区分每个处理器独有的局部状态和整个系统共享的全局状态非常重要.`kern/cpu.h`定义了大部分CPU独有的状态结构,其中包括记录了全部CPU独有状态的`struct CpuInfo`.函数`cpunum()`总是会返回调用它的CPU的ID,这个ID可以用作数组cpus的索引.使用宏`thiscpu`可以快速获取当前CPU对应的`struct CpuInfo`.

如下CPU独有的状态是我们需要关注的:
* 每个CPU的内核栈

因为多个CPU可以同时陷入内核,因此我们需要为每个CPU开辟出独立的内核栈,以防止他们互相干扰.数组`percpu_kstacks[NCPU][KSTKSIZE]`保存了所有CPU使用的内核栈.

在Lab2中,我们将`bootstack`所指向的物理内存映射到`KSTACKTOP`下方,作为BSP的内核栈.在本实验中,我们将每个CPU的内核栈映射到该区域,并使用保护页将他们分开.CPU 0对应的内核栈仍从`KSTACKTOP`向下生长.CPU1的内核栈从CPU0的内核栈底`KSTKGAP`以下开始,以此类推.具体细节可以参照`inc/memlayout.h`.

* 每个CPU的TSS和TSS descriptor

每个CPU的`task state segment,TSS`用于指定每个CPU的内核栈所在位置.CPU i的TSS保存在`cpus[i].cpu_ts`,相应地TSS descriptor定义在GDT entry`gdt[(GD_TSS0 >> 3) + i]`.`kern/trap.c`中定义的全局ts则没用了.

* 每个CPU当前运行的进程指针

因为每个CPU都可以同时运行不同的用户进程,因此我们不能再使用一个全局变量`curenv`来指向所有CPU当前运行的进程.我们将每个CPU当前运行的进程指针保存在`cpus[cpunum()].cpu_env`,或者是`thiscpu->cpu_env`.

* 每个CPU的寄存器组

所有的寄存器对于每个CPU而言,都是独有的.因此一些寄存器初始化指令,比如`lcr3(), ltr(), lgdt(), lidt()`在每个CPU上都要执行.`env_init_percpu()`和`trap_init_percpu()`即被用作在每个CPU上执行初始化动作.

除此之外,如果我们在挑战早期的实验中,设置了CPU的状态位或者其他CPU初始化操作,那么必须在每个CPU上复制这些操作.

### Exercise 3
修改`kern/pmap.c`中的`mem_init_mp()`,从`KSTACKTOP`开始映射每个CPU的内核栈,图示位于`inc/memlayout.h`.每个内核栈的大小是`KSTKSIZE`字节外加保护页`KSTKGAP`字节.完成后,代码应该可以通过`check_kern_pgdir()`检查.

### Exercise 4
`kern/trap.c`中的`trap_init_percpu()`初始化了BSP的TSS和TSS descriptor.在Lab3中这样做是正确的,但是在SMP中这样就不行了.修改这个函数,为每个CPU设置TSS和TSS descriptor.修改完成后,全局变量ts应该不再需要了.

### 测试
在完成以上两个Exercise后,通过QEMU使用4核同时运行JOS,我们应该可以看到如下打印:
```
$make qemu CPUS=4
...
Physical memory: 66556K available, base = 640K, extended = 65532K
check_page_alloc() succeeded!
check_page() succeeded!
check_kern_pgdir() succeeded!
check_page_installed_pgdir() succeeded!
SMP: CPU 0 found 4 CPU(s)
enabled interrupts: 1 2
SMP: CPU 1 starting
SMP: CPU 2 starting
SMP: CPU 3 starting
```

---

## 大内核锁

各AP在完成`mp_main()`都处于死循环中.在AP真正运行之前,我们必须首先解决SMP同时运行内核代码的条件竞争问题.其中最简单的一种办法是使用**大内核锁**.大内核锁是一个全局唯一的锁,每当有进程陷入内核,则持有该锁.当进程返回用户空间时,释放该锁.通过这种方式,我们可以解决多个CPU在运行内核代码时的竞争问题.与之相对的弊端则是,当有多个CPU同时尝试进入内核时,只有一个CPU可以进入,其他CPU不得不循环等待.

`kern/spinlock.h`声明了大内核锁`kernel_lock`,同时提供了`lock_kernel()`和`unlock_kernel()`两个函数.我们需要在如下4种位置使用大内核锁:
1. `i386_init()`: 在BSP唤醒其他AP前,获取大内核锁.
2. `mp_main()`: AP初始化之后,获取大内核锁.然后调用`sched_yield()`来调度合适的进程运行在本CPU上.
3. `trap()`: 从用户空间陷入内核后,尝试获取大内核锁.我们可以通过检查`tf_cs`的低位来判断`trap`来自用户态还是内核态.
4. `env_run()`: 在切换到用户态之前,释放大内核锁.过早或者过晚释放大内核锁都会导致竞争或者死锁.

### Exercise 5

根据上面描述的位置,使用大内核锁,从而避免SMP的竞争问题.

### 测试
目前我们还不能单独测试,在完成进程调度后,我们会测试这些代码.

### Q&&A
Q: 通过大内核锁,我们可以保证同一时间只有一个CPU运行内核代码,那么为什么还要给每个CPU单独开辟内核栈?为什么不能共用一个呢?能不能想出一个场景,即使有大内核锁的保护,但如果没有独立的内核栈,还是会出错?

A: 如果CPU a从用户态陷入内核,正在使用唯一的内核栈.此时如果有中断需要处理,我们可以看到大内核锁并没有保护中断处理.因此还是可能出现两个CPU同时在运行内核代码的情况.


---

## 循环调度

我们的下一个练习是修改JOS内核,以便在多个CPU间调度进程.调度流程如下:

`kern/sched.c`中的`sched_yield()`负责选出合适的进程来运行.它将会遍历整个`envs[]`数组查找进程状态为`ENV_RUNNABLE`的进程.如果存在上一个运行的进程则从上一个进程向后查找,如果没有则从数组头部开始.找到之后,调用`env_run()`来执行那个进程.

`sched_yield()`必须避免在两个CPU上运行同一个进程.它可以通过判断进程状态来判断进程是否正在运行.只要避免竞争问题,我们就可以避免同时调度同一进程到两个不同的CPU上.

`sys_yield()`是新实现的一个系统调用.用户进程通过调用这个函数,可以调用到内核的`sched_yield()`,从而让出CPU.

### Exercise 6
1. 完成上面描述的`sched_yield()`
2. 不要忘了修改`syscall()`来调度`sys_yield()`.
3. 确保在`mp_main()`中正确调用了`sched_yield()`.
4. 修改`kern/init.c`来创建多个进程同时运行`user/yield.c`
5. 运行`make qemu`,进程应该在彼此之间来回多次切换.
```
...
Hello, I am environment 00001000.
Hello, I am environment 00001001.
Hello, I am environment 00001002.
Back in environment 00001000, iteration 0.
Back in environment 00001001, iteration 0.
Back in environment 00001002, iteration 0.
Back in environment 00001000, iteration 1.
Back in environment 00001001, iteration 1.
Back in environment 00001002, iteration 1.
...
```
6. 运行`make qemu CPUS=2`
7. 当`yield`程序退出后,系统没有了可执行的进程.调度器会调用JOS的kernel monitor.如果不是这样,请修改代码.

### Q&&A
Q: 在我们`env_run()`中应该调用了`lcr3()`.在调用`lcr3()`前后,我们引用了变量e的内容(env_run的参数).一旦调用了`lcr3()`之后,MMU使用的寻址页表立刻变化了.但是虚拟地址e所对应的物理地址没有发生变化.这是为什么呢?即为什么可以在`lcr3()`前后都引用变量e?

A: 首先明确虚拟地址e存在于内核空间,因此内核页表存在其映射关系.其次,在创建新进程时,会拷贝内核页表到新进程的页表.因此MMU采用新进程页表进行地址转换同样可以覆盖全部内核空间.相关代码路径`env_create()-->env_alloc()-->env_setup_vm()`


Q: 不管任何时候,当内核从进程a切换到进程b.它必须确保进程a的所有寄存器被正确保存,以便一会可以继续切换到进程b运行.请问内核是如何保存的?

A: 在进程a陷入内核时,用户态所有寄存器被保存到了TrameFrame中

---
## 系统调用创建进程
现在JOS可以在多个用户进程间切换和运行,但仍然受限于内核的启动设置.我们将要实现JOS系统调用以允许用户进程创建和启动其他的新用户进程.

Unix中的`fork()`提供了进程创建原语.Unix`fork()`复制了调用进程(父进程)整个地址空间,以创建一个新的进程.在用户空间看来,两个进程之间唯一的区别在于进程id以及父进程id(getpid, getppid).在父进程中`fork()`返回子进程的id,而在子进程中`fork()`返回0.默认情况下两个进程都有自己的独立地址空间,两个进程对内存的修改都不会影响另一个线程.

我们将要实现一组不同的,更原始的JOS系统调用来创建新的用户进程.通过这些系统调用的组合,我们完全可以在用户空间时间类似Unix的`fork()`.我们有如下新的系统调用需要实现:

1. sys_exofork

该系统调用将会创建一个几乎空白的新进程.它的用户地址空间没有映射任何内容,且该进程不可运行.新进程的寄存器状态和父进程完全一致.在父进程中,`sys_exofork()`将返回新进程的id,失败的话,则返回错误代码.相应地,子进程将会返回0.不过由于子进程一开始被标记为不可运行,所以子进程实际上并不会返回,直到父进程明确标记子进程可以运行.

2. sys_env_set_status

显示设置进程状态为`ENV_RUNNABLE`或者`ENV_NOT_RUNNABEL`.当新进程的地址空间和寄存器初始化完成,父进程可以通过调用本函数标记新进程.

3. sys_page_alloc

分配一页物理内存,并在指定进程的地址空间中,映射到指定虚拟地址.

4. sys_page_map

从一个进程的地址空间,复制page映射关系到新进程的地址空间,从而完成内存共享.新进程和旧进程在各自的地址孔家,同时引用相同的物理页面.

5. sys_page_unmap

从指定进程的地址空间,取消指定虚拟地址位置的页面映射.

以上所有函数中,都可以使用0表示当前进程.这个约定是在`kern/env.c`中的`envid2env()`中实现的.

通过组合上面这些系统调用,测试程序`user/dumbfork.c`中实现了一个类似Unix的`fork()`. `dumbfork.c`创建并运行了一个新的进程.父进程和子进程之间通过`sys_yield()`来回切换.


---

### Exercise7

1. 在`kern/syscall.c`中完成上述的系统调用
2. 注意修改`syscall()`以调用上述系统调用.
3. 我们需要用到`kern/pmap.c`,`kern/env.c`,尤其是`envid2env()`.目前`envid2env()`中,参数`checkperm`填1即可.
4. 注意参数检查,若是无效参数,返回`-E_INVAL`
5. 使用`user/dumbfork`来测试代码.


### 测试
运行
```
$ ./grade-lab4 -v
```

















































----
