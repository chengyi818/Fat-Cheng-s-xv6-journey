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

实模式和保护模式的区别看起来,只是寻址空间大小发生了变化.实模式时,CS直接关联物理地址区域.而保护模式CS主要用于在GDT(global descriptor table)中查找描述符.在未激活分页功能前,通过CS和IP在GDT中对应的地址依然是物理地址.

## 对照反汇编代码调试