# 课后作业: 启动xv6
## 启动xv6
首先需要下载编译xv6源码,这个部分我们不再赘述.

## 使用gdb调试xv6
首先在xv6源码目标运行:
```
make qemu-gdb
```
然后使用一个新的terminal窗口,同样在xv6源码目录运行:
```
gdb
```

另外,我们可以通过
```
nm kernel | grep _start
```
来查看kernel的入口地址,这里我们发现是`0x0010000c`.


### 注意: 
这里gdb运行可以会报出warning,并且没有正确连接gdb server.这时我们需要根据提示去修改我们home目录下的`.gdbinit`文件.

## 练习: 栈信息分析
我们在kernel入口的位置打下断点,当gdb停在断点时,我们通过`info reg`和`x/24x $esp`来查看寄存器和栈上的信息.尝试分析栈上信息的内容,并给出简短注释.

我们可以通过bootasm.S,bootmain.c和bootblock.asm来查看bootloader的相关内容.我们的目标是分析栈上内容的信息,因此在进入栈之前,关于bootloader的运行就是问题的关键.以下是一些关键步骤:
1. 重新启动qemu和gdb,并在`0x7c00`处打下断点.单步调试bootasm.S,%esp寄存器什么时候被初始化的?
2. 刚进入`bootmain`的时候,这是栈的情况怎么样?
3. `bootmain`的第一条汇编指令是如何调整栈的?通过bootblock.asm/bootmain可以查看到这些信息.
4. 继续使用gdb调试,找到何处%eip寄存器被设置位`0x10000c`,此时栈有什么变化?

### FC备注
1. 在bootasm.S中,跳转到bootmain之前,通过指令`movl $start, %esp`设置了%esp寄存器的值.此时$start为bootloader的载入地址,即`0x7c00`.从这里开始,栈将向下生长.

2. 我们已经知道进入bootmain之前,$esp被设置为`0x7c00`.之后是`call bootmain`.call指令会将返回地址压栈.因此在刚进入bootmain时,栈顶为`0x7bfc`,栈中只有一个返回地址`0x00007c4d`.

3. 进入函数第一件事,就是将上一个函数的$ebp寄存器压栈,继而将%esp寄存器的值赋给%ebp,再调整新的%esp的值.

4. 

### 提交
输出`x/24x $esp`到文件中,并添加注释.我个人将文件保存在github`resources/HW_Boot_xv6.md`.













---