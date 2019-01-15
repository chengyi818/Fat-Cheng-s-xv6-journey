# 写时拷贝
正如之前提到的,Unix提供了`fork()`系统调用用于创建进程.`fork()`会拷贝父进程的地址空间到子进程的地址空间.

xv6的`dumbfork()`将会拷贝父进程地址空间中的全部内容到子进程的地址空间中.这部分数据拷贝是进程创建中耗费最多的部分.然而在创建新进程时,往往一个`fork()`函数后面会紧接着一个`exec()`.`exec()`会使用新的代码段替换子进程拷贝而来的代码段.比如在shell中,通常就是如此.在这种情况下,父进程花费大量时间拷贝的代码段毫无用处.

介于此,新的Unix利用虚拟内存,运行父子进程共享物理内存,直接一个进程实际修改共享内存.这项技术通常被称为**写时拷贝**.新的`fork()`将会拷贝父进程页表中的映射关系到子进程中,而不是拷贝其中的内容,同时需要将共享内存标记为只读.当两个进程有进程尝试对共享内存写入时,该进程将会发生`page fault`.此时,内核立刻意识到该进程正确操作一块写时拷贝页面,内核会为该进程创建一个新的可写的页面.正是通过这种方式,我们推迟了页面实际拷贝的时间点,只有到真正写入时才会执行拷贝.这种优化使得`fork()+exec()`的消耗大幅降低,在很多时候,创建进程仅仅需要拷贝一个页面即可(当前的进程栈).

在本部分,我们将会在用户空间库中,实现一个类似Unix的`fork()`.在用户空间实现`fork()`的好处在于可以保持内核尽可能的简单,且不容易出错.用户程序完全可以根据自己的需要来组合,定义自己的`fork()`实现.比如`dumbfork()`或者真正类似于Unix`fork()`.

编者按: 本质上这种思想和Unix思想一脉相承,通过提供小而简单的工具,又用户来自由组合使用.提供机制而不是策略.

---

## 用户态page fault处理
用户态写时拷贝的实现首先需要知道在写保护的页面上发生了`page fault`,用户态在许多情况下都可能触发`page fault`,写时拷贝只是其中的一种情况.

延迟分配是Unix系统常用的策略.比如在进程创建时,大多数Unix系统仅为新进程分配了一个页面的函数栈,当进程栈消耗一空并触发`page fault`后,内核会再次分配并映射物理内存页面.Unix内核必须清楚在进程空间的不同位置发生`page fault`时,所需要采取的动作.比如在函数栈发生了`page fault`,需要分配并映射物理页面.在BSS段发生`page fault`,需要分配和映射物理页面,并将该页面全部置0.在支持按需分配二进制代码的系统中,在代码段的`page fault`,处理分配映射物理内存外,还需要从磁盘上读取对应的二进制文件.

以上所有信息都是内核所需要知悉的.不同于传统的Unix实现方法,我们需要在用户空间决定如何处理用户空间的各种`page fault`.这种设计方式使得程序定义其内存区域时更加灵活.在后面的练习中,我们将使用用户态的`page fault`处理来映射和访问磁盘文件系统.

## 设置`page fault`处理
为了处理进程的page fault,用户进程需要向JOS内核注册自己的page fault处理程序.用户进程通过`sys_env_pgfault_upcall`来注册`page fault`处理程序.在`struct Env`中,添加了一个新的成员变量`env_pgfault_upcall`记录`page fault`处理程序

### Exercise8
实现`sys_env_set_pgfault_upcall`.在使用`envid2env()`时,确保打开了权限检查,这是比较危险的一个系统调用.

---

## 用户空间的普通函数栈和异常栈
用户进程在正常执行过程中,使用普通函数栈.esp寄存器从`USTACKTOP`开始,数据保存在从`USTACKTOP-PGSIZE`到`USTACKTOP-1`之间的页面上.当发生page fault时,内核将重启用户进程,并在异常栈中执行用户指定的`page fault`处理程序.本质上,我们需要是JOS内核代替用户空间自动完成**用户空间栈切换**,这非常类似于x86处理器从用户态切换到内核态时,此时函数栈已经从用户栈切换到内核栈.

JOS的用户异常栈同样是一个页的大小,栈顶位于`UXSTACKTOP`,所以有效的存储空间是从`UXSTACKTOP-PGSIZE`到`UXSTACKTOP-1`.在异常栈上运行时,用户态的page fault handler可以使用JOS的系统调用来分配映射新的页面,从而修复page fault错误.当用户态的page fault handler返回时,通过汇编语言重新回到普通的函数栈中.

用户进程要想支持用户态的page fault handler,需要为每个进程分配异常栈.我们可以通过part A介绍的`sys_page_alloc()`函数来分配.

## 调用用户态page fault handler
为了从用户态处理page fault,我们需要修改`kern/trap.c`中的page fault handler.我们将page fault时的用户状态称为**trap-time**.

如果没有page fault handler注册,JOS内核会销毁进程.否则,内核会在异常栈上准备好一个`trap frame`,具体定义可以参考`int/trap.h`中的`struct Trapframe`.
```
                    <-- UXSTACKTOP
trap-time esp
trap-time eflags
trap-time eip
trap-time eax       start of struct PushRegs
trap-time ecx
trap-time edx
trap-time ebx
trap-time esp
trap-time ebp
trap-time esi
trap-time edi       end of struct PushRegs
tf_err (error code)
fault_va            <-- %esp when handler is run

```

接着,内核会回复用户进程的运行,page fault handler将会在异常栈上运行.`fault_va`是引起`page fault`的地址.当异常发生时,用户进程已经运行在异常栈上,此时相当于page fault handler发生异常,我们必须在当前的`tf->tf_esp`下方压入新的栈帧.首先压入一个空的32bit,接着是一个`struct UTrapframe`.

通过判断`tf->tf_esp`的值是否位于`UXSTACKTOP-PGSIZE`到`UTSTACKTOP-1`,可以知道`tf->tf_esp`是否已经处于异常栈中.

### Exercise9
实现`kern/trap.c`中的`page_fault_handler()`,将page fault分派给用户态handler处理.在使用异常栈时需要考虑适当的保护措施,比如当用户进程超出异常栈空间的话,要怎么处理.

### Exercise10
下面我们需要实现汇编代码以实现从page fault handler,恢复到引起page fault的位置.这段汇编代码将会通过`sys_env_set_pgfault_upcall()`注册到内核中.

实现`lib/pfentry.S`中的`_pgfault_upcall`.其中比较有趣的地方在于如何回到触发page fault的指令位置.我们将从用户空间不经内核直接跳转过去.难点在于如何同时切换函数栈以及重新加载eip.


### Exercise11
最后,我们需要在用户空间`lib/pgfault.c`中实现库函数`set_pgfault_handler()`

### 测试
1. 通过命令`make run-faultread`, 运行用户程序`user/faultread`.
```
...
[00000000] new env 00001000
[00001000] user fault va 00000000 ip 0080003a
TRAP frame ...
[00001000] free env 00001000
```
2. 通过命令`make run-faultdie`,运行用户程序`user/faultdie`
```
Run user/faultdie. You should see:

...
[00000000] new env 00001000
i faulted at va deadbeef, err 6
[00001000] exiting gracefully
[00001000] free env 00001000
```

3. 通过命令`make run-faultalloc`,运行用户程序`user/faultalloc`
```
Run user/faultalloc. You should see:

...
[00000000] new env 00001000
fault deadbeef
this string was faulted in at deadbeef
fault cafebffe
fault cafec000
this string was faulted in at cafebffe
[00001000] exiting gracefully
[00001000] free env 00001000
```
如果这里你只看到`this string`这一行,意味着我们没有正确处理`page fault`递归的情况.


4. 通过命令`make run-faultallocbad`,运行用户程序`user/faultallocbad`
```
...
[00000000] new env 00001000
[00001000] user_mem_check assertion failure for va deadbeef
[00001000] free env 00001000
```
确保明白了为什么`user/faultalloc`和`user/faultallocbad`打印不同.
































---
