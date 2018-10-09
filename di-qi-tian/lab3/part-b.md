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