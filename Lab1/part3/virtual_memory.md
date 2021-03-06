# 虚拟内存
之前我们查看过Bootloader的link address和Load address,它们是一致的,均为0x7c00.而我们的kernel要复杂一些.它的load address是0x100000,而执行入口是0x10000c.我们是在`kern/kernel.ld`中规定了kernel的load address和e_entry.

通常操作系统都会将内核载入到一个很高的虚拟地址,比如0xf0100000.从而将虚拟地址的低地址区间留给用户空间.我们后边会探讨这样做的原因.

当然许多计算机并没有3GB多的内存,也就不存在0xf0100000这样的物理内存地址.因此我们当然不能将内存载入到0xf0100000这样的物理地址.通常我们是这样的做的,利用处理器的内存管理单元将虚拟地址0xf0100000映射到物理地址0x00100000.这样我们既在虚拟地址的低地址部分留给了用户空间,也能够将内存紧贴着BIOS顺利载入物理内存.

事实上,在Lab2中我们会将物理内存0x00000000~0x0fffffff映射到虚拟内存0xf0000000~0xffffffff,整体上移0xf0000000.这也是为什么JOS只能使用256MB物理内存的原因.

当前,我们仅仅映射了物理内存前4MB空间.我们是通过`kern/entrypgdir.c`静态生成了映射表.在`kern/entry.S`将`CR0_PG`标志位置位前,内存地址是线性地址.在JOS中也表示物理地址,因为`boot/boot.S`中将线性地址映射为物理地址.

一旦`CR0_PG`被置位,内存地址将变为虚拟地址.虚拟地址可以通过硬件内存管理单元转换为物理地址.`entry_pgdir`将虚拟地址0xf0000000~0xf0400000转换到0x00000000~0x00400000
.也将虚拟地址0x00000000~0x00400000映射到同一块物理内存0x00000000~0x00400000.这种两块虚拟内存映射同一片物理内存的情形,后面我们还会再次遇到.

如果这时我们操作不再这两块虚拟地址区域的地址,因为地址无法转换,即无法找到对应的物理地址.如果没有设置相应的错误处理,那么QEMU模拟器将会崩溃.在我们定制过的QEMU中,QEMU将会无限重启.

## Exercise 7
1. 使用QEMU和GDB调试JOS内核,并暂停在`movl %eax, %cr0`之前,此时检查0x00100000和0xf0100000处的内存值.
2. 使用GDB命令`stepi`单步执行指令,再次检查上述地址内存值.
3. 如果没有虚拟地址映射,那么在kernel中哪条指令会报错.注释掉`kern/entry.S`中的`movl %eax, %cr0`,再次调试内核试试看.


1. 在`movl %eax, %cr0`后,0xf0100000处内存值并未改变,在`mov $relocated, %eax`后,该内存值映射到0x00100000.

3. jmp指令设置了CS和IP寄存器,当CS和IP再次查找指令时,虚拟机报错退出.