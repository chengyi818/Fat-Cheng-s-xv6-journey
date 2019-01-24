# x86模拟器
在我们的开发中,我们将使用QEMU模拟器来进行调试开发.使用模拟器,我们可以使用GDB的一系列调试命令来调试我们的代码.

## 编译jos
在jos源码目录输入`make`,将会编译jos代码.

编译完成后,bootloader生成位置`obj/boot/boot`,Kernel生成位置`obj/kern/kernel.img`

## 启动qemu
在jos源码目录输入`make qemu`,将会直接启动QEMU来运行我们的jos系统.

我们将会看到输出如下:

```
Booting from Hard Disk...
6828 decimal is XXX octal!
entering test_backtrace 5
entering test_backtrace 4
entering test_backtrace 3
entering test_backtrace 2
entering test_backtrace 1
entering test_backtrace 0
leaving test_backtrace 0
leaving test_backtrace 1
leaving test_backtrace 2
leaving test_backtrace 3
leaving test_backtrace 4
leaving test_backtrace 5
Welcome to the JOS kernel monitor!
Type 'help' for a list of commands.
K> 
```

Booting from Hard Disk...之后的输出,都是我们的jos内核打印的.

## 运行jos
目前我们的jos仅仅支持两条命令`help`和`kerninfo`,看看会输出什么.
