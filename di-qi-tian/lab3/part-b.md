<!-- toc -->

# 页异常,中断点, 系统调用
现在JOS已经有了基本的异常处理功能,下面我们将继续增强异常处理功能.

## 处理page fault
page fault异常,中断号为14(T_PGFLT),是一个非常重要的异常.当CPU发生page fault的时候,CPU将线性地址(等同于虚拟地址)保存到了寄存器CR2.在`trap.c`中,`page_fault_handler()`函数是用来处理`page fault`异常的.

## Exercise 5
修改`trap_dispatch()`函数,将`page fault`异常交由`page_fault_handler()`处理.

### Exercise 5 测试
通过运行`faultread`, `faultreadkernel`, `faultwrite`和`faultwritekernel`进行测试.

### Tips
我们可以通过命令`make run-x`来运行用户空间程序x.比如`make run-hello`可以运行用户程序`hello`.

---

## 中断点异常breakpoint Exception
中断点异常,中断号为3(T_BRKPT),允许调试者在程序中插入一个特殊的1字节指令`int3`.当CPU执行到该执行时,将会触发中断点异常.在JOS,我们将会稍微扩展下这个异常,当该异常触发时,将会调用`kernel monitor`.用户空间`lib/panic.c panic()`的实现中,在打印了一些崩溃信息后,主动执行了`int3`指令.

## Exercise 6
修改`trap_dispatch()`函数,使中断点异常调用`kernel monitor`.

### Exercise 6 测试
通过运行`breakpoint`进行测试

### 挑战
1. 修改JOS kernel monitor,使得可以从当前断点的位置**继续**执行,即如果`kernel monitor`是由中断点异常调用的,可以返回用户空间继续执行.这样我们就可以单步调试程序了.我们需要理解`EFLAGS`寄存器中的某些比特位以便实现单步调试.

2. 如果你比较喜欢挑战,可以尝试反汇编单步调试的代码.结合lab 1中获取的symbol table,这基本已经是一个完整的内核调试器了.

### 问题
3. 中断异常的测试用例既可以触发中断点异常处理(3)也可以触发通用保护异常处理(13),这取决于你如何设置IDT.为什么?如何正确设置IDT?为什么不正确地设置IDT会导致触发通用保护异常处理?

    考虑IDT中的权限问题,若用户空间没有权限使用中断点中断处理,会转为触发通用保护异常处理.

4.  这些机制的意义在什么地方?特别是考虑到`user/softint`的作用.
    权限控制,只有符合权限的调用者才能触发特定的异常处理.
    
---

## 系统调用system call
用户程序通过系统调用对内核发起请求.当用户程序调用一个系统调用,CPU将进入内核态,同时CPU和内核共同完成了用户态的保存,接着内核将执行系统调用对应的代码,最后返回用户态,恢复用户程序的执行.用户程序是如何陷入内核以及如何通知内核所调用的系统调用,则根据不同的系统有所不同.

在JOS中,我们通过`int`指令来触发CPU中断.我们规定系统调用中断号为`$0x30`,即`T_SYSCALL`.我们必须正确设置IDT,以允许用户程序产生该中断.值得一提的是,中断`0x30`不会由硬件产生,所以运行用户程序产生该中断,并不会引发歧义.

用户程序指定的系统调用号和参数将会通过寄存器来传递.这样做的好处是,内核不需要在用户空间的堆栈和指令流中搜寻参数了.我们使用`%eax`保存系统调用号,使用`%edx, %ecx, %ebx, %edi, %esi`等5个寄存器分别保存参数,因此我们至多支持5个参数.完成系统调用后,内核将返回值保存到`%eax`寄存器.调用系统调用的汇编指令已经在`lib/syscall.c syscall()`完成.请认真阅读,并确保理解了`syscall()`的实现.

## Exercise 7
在内核中增加中断向量`T_SYSCALL`的处理.我们需要修改`kern/trapentry.S`和`kern/trap.c trap_init()`.我们还需要修改`trap_dispatch()`函数,使用`kern/syscall.c syscall()`函数去处理系统调用中断,然后将`%eax`中设置返回值传递会用户进程.最后我们还需要在`kern/syscall.c`中实现`syscall()`函数.如果系统调用号是非法的,确保`syscall()`函数返回`-E_INVAL`.我们需要阅读并理解`lib/syscall.c`(特别是其中的内嵌汇编部分),以确保我们对系统调用接口的理解.我们还需要为`inc/syscall.h`中列出的所有系统调用,调用相应的内核处理函数.

### Exercise 7 测试
通过命令`make run-hello`来运行用户程序`user/hello`.首先应该在console中打印出`hello, world`,然后在用户模式下触发`page fault`.如何结果不符合预期,或许是我们的实现有问题.

### 挑战
使用`sysenter`和`sysexit`指令实现系统调用,而不是使用`int 0x30`和`iret`.

`sysenter`/`sysexit`指令由英特尔设计,比`int/iret`更快.他们通过使用寄存器而不是堆栈,并预测分段寄存器的使用方式来实现这一点.这些指令的细节可以在英特尔参考手册第2B卷中找到.

在JOS中添加对这些指令的支持的最简单方法是在`kern/trapentry.S`中添加一个`sysenter_handler`,它保存了足够的关于用户空间的信息以返回到用户空间,设置了内核环境,将参数传递给`syscall()`并直接调用`syscall()`.当`syscall()`返回,设置并运行`sysexit`指令.您还需要在`kern/init.c`中添加代码,以设置必要的`model specific`寄存器(MSRs).《AMD体系结构程序员手册》第2卷中的第6.1.2节和《英特尔参考手册》第2B卷中关于`SYSENTER`参考文件对相关MSRs进行了详细描述.你可以在[这里](http://ftp.kh.edu.tw/Linux/SuSE/people/garloff/linux/k6mod.c)找到`inc/x86.h wrmsr`是如何写入这些MSRs的.

最后,必须更改`lib / syscall.c`以支持使用`sysenter`调用系统调用.























































---