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

## 为什么