# The Boot Loader
## 简介
在电脑硬盘被划分为了512字节的区域,我们称之为扇区.每次读写硬盘都是以扇区的整数倍为单位.如果一个磁盘是可启动的(bootable),那么它的第一个扇区我们称之为boot扇区,这也是bootloader存储的位置.

当BIOS发现一个可启动扇区后.首先,它会将boot扇区中的512字节,载入到物理内存0x7c00~0x7dff的位置.然后,跳转到0x7c00开始执行.

和BIOS在物理内存中的地址一样,虽然没有什么规定强制要求,但是载入的位置已经是现在计算机约定俗成的了.

另外我们稍微说明下从CD-ROM也就是光盘启动.光盘启动是计算机发展到20世纪90年代才具备的能力.光盘中一个扇区大小为2048字节.此时的BIOS可以载入一个远比512字节大的BootLoader到内存中.更多详细信息,请参考[El Torito" Bootable CD-ROM Format Specification](https://pdos.csail.mit.edu/6.828/2017/readings/boot-cdrom.pdf)

## JOS BootLoader
JOS还是使用了传统的从硬盘启动的机制,也就是说我们的Bootloader仅有可怜的512字节.我们的Bootloader是有两部分组成的:
1. 汇编代码: boot/boot.S
2. C代码: boot/main.c
仔细看看这两份文件,确保理解了这部分内容.

## JOS Bootloader功能
* 首先,将CPU模式从16位实模式转为32位保护模式

显然在16位模式下,可寻址范围仅有1MB.只有切换为32位模式,我们才能够使用1MB以上的内存.

关于保护模式可以参考[PC Assembly Language](https://pdos.csail.mit.edu/6.828/2017/readings/pcasm-book.pdf)的1.2.7,1.2.8小节.当前我们仅需知道在保护模式下,通过CS,IP寄存器查找物理内存的方法会发生变化即可.

* 其次,Bootloader将从IDE硬盘中将内核载入内存.

如果对这些IO指令感兴趣,可以参考[the 6.828 reference page](https://pdos.csail.mit.edu/6.828/2017/reference.html)中的*IDE hard drive controller*部分.我们并不需要太过深入这个部分,虽然设备驱动是操作系统开发中,非常重要的部分.但是从整个操作系统的结构来看,它也是最无趣的部分了.

### 疑问
看起来boot.S中已经将CPU模式从实模式切换为32位保护模式,为何bootmain中还能够直接操作物理地址?

看注释是说在readseg时,还没激活分页功能?

实模式和保护模式的区别看起来只是寻址方式发生了变化.实模式时,CS直接关联物理地址区域.而保护模式CS主要用于在GDT(global descriptor table)中查找描述符.在未激活分页功能前,通过CS和IP在GDT中对应的地址称为线性地址,依然是物理地址.

注意区分: 逻辑地址, 线性地址, 物理地址.

## GDB简要命令
在`obj/kern/kernel.asm`中是内核代码的反汇编结果.

* b *0x7c00
在0x7c00处,加入断点.

* continue/c
GDB执行指令,直到遇到断点或者结果.

* si
GDB单步执行指令

* si N
GDB一次执行N条指令

* x/i
查看下一条要执行的指令

* x/Ni
查看下N条要执行的指令

* x/Ni ADDR
查看ADDR后,接着要执行的N条指令

* 回车键
重复上一条指令

## Exercise 3
看完两份源码之后,`obj/boot/boot.asm`是我们整个BootLoader的反汇编代码.我们可以一遍对照着反汇编代码,一边用GDB来查看对应的指令执行过程.

试着回答几个问题:
1. CPU什么时候切换到32位模式的,是什么造成了这样的切换
将CR0寄存器中的CR0_PE_ON标志位置位,意味着真正切换到了32位模式.
2. BootLoader执行的最后一条指令是什么,Kernel执行的第一条指令是什么?
BootLoader执行的最后一条指令是:
Kernel执行的第一条指令是:
3. Kernel第一条指令的地址?
4. BootLoader是怎么知道kernel的大小,什么地方有这样的信息
Kernel是一个ELF文件,那么在ELF头部信息中,尤其是programHead中就带有了ELF文件大小的信息.














---