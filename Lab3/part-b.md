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

最后,必须更改`lib / syscall.c`以支持使用`sysenter`调用系统调用.以下是`sysenter`指令的可能寄存器布局:
```
	eax                - syscall number
	edx, ecx, ebx, edi - arg1, arg2, arg3, arg4
	esi                - return pc
	ebp                - return esp
	esp                - trashed by sysenter
```

当使用GCC的内嵌汇编程序直接加载值时,会自动保存寄存器值.不要忘了push和pop你正在使用的其他寄存器,或者告诉内嵌汇编程序你正在使用它们.内嵌汇编程序不支持保存`%ebp`,因此您需要添加代码来保存和恢复它.通过使用`leal after_sysenter_label, %%esi`这样的指令,可以将返回地址放入`%esi`中.

请注意,这仅支持4个参数,因此您需要保留旧的系统调用方法来支持5个参数系统调用.此外,由于这种快速调用凡是不会更新当前用户空间的的`TrapFrame`,因此不适用于我们在以后的Lab中添加的一些系统调用.

在下一个实验中我们将启用异步中断,你可能需要重新修改这部分代码.具体来说,当返回到用户进程后,你需要允许中断,而`sysexit`并没有为你设置.

---
## 用户模式模式
用户程序的入口位于`lib/entry.S`.在经过一些设置后,主要是函数入参,将会调用位于`lib/libmain.c`的`libmain()`函数.我们需要修改`libmain()`函数来正确地初始化全局变量`thisenv`,指向当前进程位于`struct envs[]`中的结构体.`lib/entry.S`已经定义了`envs`,并指向`UENVS`.

提示: 查看`inc/env.h`,并使用`sys_getenvid`.

`libmain()`函数将调用`umain()`.在`hello`程序的情况下,它的定义位于`user/hello.c`.当打印完`hello, world`之后,`hello`程序尝试获取`thisenv->env_id`.这就是`Exercise 7`之后,我们的程序`page fault`的原因.在正确设置全局变量`thisenv`之后,`hello`程序应该会一切正常.如果仍然失败,那么可能是由于在Part A `pmap.c`中,UENVS设置的不对.

## Exercise 8
将上述所需的代码添加到用户库中,然后启动内核.你应该可以看到`hello`程序首先打印出`hello, world`,然后打印出`i am environment 00001000`.
然后`user/hello`将尝试调用`sys_env_destroy()`(`lib/libmain.c`和`lib/exit.c`)来退出.由于内核目前只支持一个用户环境,它应该报告它已经销毁了唯一的环境,然后进入`kernel monitor`.

---

## Page faults和内核保护
内存保护是操作系统的一个重要特性,确保一个程序中的错误不会损坏其他程序或操作系统本身.

操作系统通常依赖硬件支持来实现内存保护.操作系统向硬件通知哪些虚拟地址有效,哪些无效.当一个程序试图访问一个无效的地址或它没有权限访问的地址时,处理器会在导致故障的指令处停止该程序,然后携带关于所尝试操作的信息陷入到内核中.如果故障是可修复的,内核可以修复它,让程序继续运行.如果故障无法修复,则程序无法继续,因为它永远不会通过导致故障的指令.

作为可修复故障的一个例子,我们来看下具有自动扩展特性的函数栈.在许多系统中,内核最初仅为函数栈分配了一个页面,然后如果程序错误访问函数栈更下面的页面,内核将自动分配这些页面并让程序继续运行.通过这种方式,内核仅分配程序所需的栈内存,但是程序会有一种函数栈任意大的假象.

系统调用对内存保护的支持引起了一个有趣的问题.大多数系统调用接口允许用户程序将指针传递到内核.这些指针指向要读取或写入的用户缓冲区.内核然后在执行系统调用时引用这些指针所指向的内容.这将会导致两个问题:

1. 内核中的page fault要比用户程序中的page fault严重得多.如果内核页面在操作自己的数据结构时page fault,那是`kernel panic`,并且错误处理程序应该使内核(因此整个系统)死机.但是当内核引用用户程序传递给它的指针时,它需要一种方式来记住这些引用导致的page fault实际上相当于用户程序导致的.

2. 内核通常比用户程序有更高的内存权限.用户程序传递给系统调用的指针可能指向内核可以读写但程序不能读写的内存.内核必须注意,不要被去操作这样的指针,因为这可能会泄露隐私信息或者破坏内核的完整性.

由于这两个原因,内核在处理用户程序提供的指针时必须非常小心.

现在,我们将使用一个机制来解决这两个问题,该机制会仔细检查从用户空间传递到内核中的所有指针.当程序向内核传递指针时,内核将检查地址是否属于用户的地址空间,以及页表是否允许内存操作.

因此,内核永远不会因为引用用户提供的指针而产生`page fault`.如果内核出现`page fault`,它应该会死机并终止.

## Exercise 9
修改`kern/trap.c`,如果在内核模式下出现`page fault`,内核将会`panic`.

提示: 可以通过检查`TF_cs`的低位,来确定`fault`是发生在用户模式还是内核模式.

阅读`kern/pmap.c`中`user_mem_assert()`函数,并在同一文件中实现`user_mem_check()`.

修改`kern/syscall.c`,对系统调用传入的参数进行安全检查.

运行用户程序`user/buggyhello`.用户进程应该被销毁而内核不受影响.打印内容应该如下:
```
	[00001000] user_mem_check assertion failure for va 00000001
	[00001000] free env 00001000
	Destroyed the only environment - nothing more to do!
```

最后,修改`kern/kdebug.c`中的`debuginfo_eip()`函数,使用`user_mem_check`对`usd`,`stabs`,`stabstr`.如果现在运行`user/breakpoint`,应该能够在`kernel monitor`运行`backtrace`,同时在内核因`page fault`而死机之前看到`backtrace`追溯到`lib/libmain. c`.导致此`page fault`的原因是什么？你不需要修复它，但是你应该理解它为什么会发生。

---
## Exercise 10
注意,我们刚刚实现的相同机制也适用于恶意用户应用程序(如`user/evillhello`).

运行`user/evilhello`,用户进程应该被销毁,并且内核正确运行.我们将会看到如下打印信息:
```
	[00000000] new env 00001000
	...
	[00001000] user_mem_check assertion failure for va f010000c
	[00001000] free env 00001000
```






































---