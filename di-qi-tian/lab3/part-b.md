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
