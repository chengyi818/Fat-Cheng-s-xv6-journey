# BIOS
本小节,我们需要了解jos是如何启动的.

我们打开两个terminal窗口,都进入jos目录.在第一个输入`make qemu-gdb`,在第二个输入`make gdb`

输出如下:
```
athena% make gdb
GNU gdb (GDB) 6.8-debian
Copyright (C) 2008 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "i486-linux-gnu".
+ target remote localhost:26000
The target architecture is assumed to be i8086
[f000:fff0] 0xffff0:	ljmp   $0xf000,$0xe05b
0x0000fff0 in ?? ()
+ symbol-file obj/kern/kernel
(gdb) 
```

我们可以看到执行的第一条语句是:
```
[f000:fff0] 0xffff0:	ljmp   $0xf000,$0xe05b
```

我们可以分析第一条语句,得出几条结论:
1. 初始指令的位置位于0x000ffff0,位于BIOS所在64KB的最顶部了.后面仅剩16个字节.
2. PC起始时,CS=0xf000,IP=0xfff0.CS和IP寄存器是x86最重要的两个寄存器了,它们标识了CPU当前要执行的指令地址.CS为代码段寄存器,IP为指令指针寄存器.
3. 第一条指令是jmp指令,而且目标地址是CS=0xf000,IP=0xe05b.

## 为什么第一条语句是这样?
这个问题涉及到了x86的设计.BIOS是静态写死载入到0x000f0000~0x000fffff中.那么第一句执行0x000ffff0保证了BIOS代码首先得到执行.毕竟在刚加电的时候,内存中根本没有其他可以执行的指令.

QEMU模拟器启动之后,将它的BIOS载入到0x000f0000~0x000fffff,同时将CS=0xf000,IP=0xfff0.继而执行相应位置的指令.

## 寄存器换算物理地址
在实模式中的寄存器,物理地址换算公式是这样的:
```
物理地址 = 16*CS + IP
```

## Exercise 2
```
使用*si*指令,追踪BIOS代码并尝试理解指令的含义.没有必要过度深究,只要知道大概即可.
```
参考资料:[Phil Storrs PC Hardware book](http://web.archive.org/web/20040404164813/members.iweb.net.au/~pstorr/pcbook/book2/book2.htm)

## BIOS目的
1. 建立中断向量表
2. 初始化硬件设备,如VGA显示
3. 初始化PCI总线及其他硬件设备.
4. 查找bootloader所在的存储设备,如磁盘,硬盘,光盘等.
5. 载入bootloader,并将控制权转移给bootloader.
