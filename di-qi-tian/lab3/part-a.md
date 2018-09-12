# 用户进程和异常处理

在文件`inc/env.h`中,JOS定义了`Env`结构体.JOS使用这个结构体来表示进程.

在Lab3,我们只会创建一个进程.在Lab4中,我们将会涉及到进程fork的问题.

正如在`kern/env.c`中所看到的,JOS使用三个全局变量来管理进程.
```
struct Env *envs = NULL;		// All environments
struct Env *curenv = NULL;		// The current env
static struct Env *env_free_list;	// Free environment list
```

我们来稍加解释,`envs`将指向一个`Env`数组.JOS最大支持NENV(定义在`inc/env.h`)个进程数.通常运行的进程远不到这个数目.在`env_init()`的函数中,我们将会初始化`envs`,在其中填充`Env`结构体.

JOS使用`env_free_list`链表来管理未使用的`Env`结构体.`curenv`则表示当前正在运行的`Env`结构体.在第一个进程真正运行前,`curenv`置NULL.

## 进程结构体Environment State

```
struct Env {
	struct Trapframe env_tf;	// Saved registers
	struct Env *env_link;		// Next free Env
	envid_t env_id;			// Unique environment identifier
	envid_t env_parent_id;		// env_id of this env's parent
	enum EnvType env_type;		// Indicates special system environments
	unsigned env_status;		// Status of the environment
	uint32_t env_runs;		// Number of times environment has run

	// Address space
	pde_t *env_pgdir;		// Kernel virtual address of page dir
};
```
我们来看看进程的管理结构体,下面是各字段的解释:

### env_tf
`struct Trapframe`定义在`inc/trap.h`中,该结构体保存了进程的寄存器值.比如A进程不运行时,内核将在进程A的Env结构体中,保存A运行时的寄存器.当进程A需要恢复运行时,可以通过`Trapframe`来恢复.

### env_link
前面提到JOS通过全局变量`env_free_list`来管理空闲的`Env`结构体.`env_link`即指向下一个空闲`Env`.

### env_id
```
 +1+---------------21-----------------+--------10--------+
 |0|          Uniqueifier             |   Environment    |
 | |                                  |      Index       |
 +------------------------------------+------------------+
                                       \--- ENVX(eid) --/

```
如图,一个`env_id`主要由两部分组成,`Uniqueifier`和`Environment Index`.其中`Uniqueifier`每次都会发生变化,而`Environment Index`则代表了使用了`envs`数组中的index.因此即使`Env`结构体被重复使用,进程的`env_id`值也不会相同.


### env_parent_id
该字段保存了进程的父进程的id.通过这种方式,进程就被组成为一个进程树.当需要判断进程是否有权限进行某个操作时,通过进程树可以进行权限判断.

### env_type
该字段用于标识特殊的进程.大部分情况下,该字段为`ENV_TYPE_USER`.后面的实验中,将会介绍其他的进程类型

### env_status
这个就是常说的进程状态了.在JOS中,进程有如下状态.
* ENV_FREE
	进程未激活,即Env结构体在env_free_list链表中
	
* ENV_RUNNABLE
	进程准备就绪,正等待内核调度运行
	
* ENV_RUNNING
	进程正在运行 

* ENV_NOT_RUNNING
	进程阻塞,此时进程已激活.但是还未准备就绪.比如正在等待IO.
	
* ENV_DYING
	僵尸进程,当僵尸进程陷入内核后,将被内核释放.
	
### env_pgdir
进程使用的页表


与Unix进程类似,JOS进程同样拥有线程和地址空间的概念.线程主要是由`env_tf`来定义,而地址空间则由`env_pgdir`来定义.为了运行一个进程,内核必须设置好`env_tf`和`env_pgdir`.

JOS中的`Struct Env`和xv6中的`struct proc`类似.区别在于xv6中每个proc都拥有自己的内核栈,多个进程可以同时陷入内核.而JOS同一时间只能有个一个进程陷入内核,因此可以共用一个内核栈.

---

## 分配进程管理数组
在Lab2中,我们通过`mem_init()`来初始化`pages[]`数组.该数组用于管理物理页内存的分配.现在我们需要修改`mem_init()`来分配一个相似的`envs`数组,来管理进程结构体.

### Excercise 1
修改`kern/pmap.c`中的`mem_init()`函数来分配和初始化`envs`数组.数组中包含了NENV个Env结构体.和`pages`数组类似,`envs`数组同样需要映射到`UENVS`(定义在inc/memlayout.h),这样用户进程才能够访问这个数组.

修改完成后,代码应该可以通过`check_kern_pgdir()`的检查.

---

## 创建和运行进程 
下面我们将完善`kern/env.c`以运行一个用户进程.

此时,我们还没有文件系统,因此用户进程所使用的二进制文件将被静态打包到Kernel文件中.Lab3的`GNUmakefile`将在`obj/user`目录中生成一系列二进制文件.如果我们再稍微深入一点,在`kern/Makefrag`中,我们看到首先定义了变量`KERN_BINFILES`,随后通过命令`ld -b binary $(KERN_BINFILES)`将这些二进制文件打包到内核文件中.我们再来看看内核文件的符号表`obj/kern/kernel.sym`,我们将看到如下的一些符号:

```
f0119356 D _binary_obj_user_hello_start
f0120b56 D _binary_obj_user_buggyhello_start
f0120b56 D _binary_obj_user_hello_end
f012835e D _binary_obj_user_buggyhello2_start
f012835e D _binary_obj_user_buggyhello_end
f012fb7e D _binary_obj_user_buggyhello2_end
f012fb7e D _binary_obj_user_evilhello_start
f0137382 D _binary_obj_user_evilhello_end
f0137382 D _binary_obj_user_testbss_start
f013eb9e D _binary_obj_user_divzero_start
f013eb9e D _binary_obj_user_testbss_end
f01463b6 D _binary_obj_user_breakpoint_start
f01463b6 D _binary_obj_user_divzero_end
f014dbbe D _binary_obj_user_breakpoint_end
f014dbbe D _binary_obj_user_softint_start
f01553c2 D _binary_obj_user_badsegment_start
f01553c2 D _binary_obj_user_softint_end
f015cbca D _binary_obj_user_badsegment_end
f015cbca D _binary_obj_user_faultread_start
f01643ce D _binary_obj_user_faultread_end
f01643ce D _binary_obj_user_faultreadkernel_start
f016bbda D _binary_obj_user_faultreadkernel_end
f016bbda D _binary_obj_user_faultwrite_start
f01733e2 D _binary_obj_user_faultwrite_end
f01733e2 D _binary_obj_user_faultwritekernel_start
f017abee D _binary_obj_user_faultwritekernel_end
```
正是通过这些符号,内核可以访问这些二进制文件.

### Exercise 2
在`kern/init.c`中的`i386_init()`函数将会运行创建用户进程并运行二进制文件.当然此时,它们还是一个半成品,你需要完成`env.c`中的如下函数:

* env_init()
	初始化envs数组中的所有Env结构体,同时把它们加入env_free_list管理.同时,我们会调用`env_init_percpu()`来设置CPU的分段硬件.
	
* env_setup_vm()
	为用户进程创建一个页目录并初始化
	
* region_alloc()
	为用户进程分配和映射物理内存

* load_icode()
	解析ELF文件,并将其内容载入用户进程空间
	
* env_create()
	调用env_alloc创建一个用户进程,然后调用load_icode载入ELF镜像
	
* env_run()
	运行一个指定的用户进程
	
备注:
在完成上述编码时,使用`cprintf`的`%e`命令可以打印出错误码所对应的错误信息,这有助于我们调试代码.举例如下:
```
r = -E_NO_MEM;
panic("env_alloc: %e", r);

输出:
env_alloc: out of memory
```

### 用户进程调用流程
下图是用户进程调用示意图,请确保理解了下图的每个步骤.
```
start (kern/entry.S)
i386_init (kern/init.c)
	cons_init
	mem_init
	env_init
	trap_init (still incomplete at this point)
	env_create
	env_run
		env_pop_tf
```

当我们完成了Exercise 2的编码之后,我们可以通过QEMU来运行.程序将一直执行`hello`程序,直接该程序调用`int`系统调用.此时JOS还没有配置硬件中断处理,因此用户空间无法调用内核.当CPU遇到无法处理的系统调用中断时,将会产生一个通用保护异常.紧接着CPU会发现通用保护异常也无法处理,将会产生一个`double fault`异常,这个异常当然也无法处理.CPU最终放弃处理,并抛出`triple fault`.

通常此时CPU将会重置,系统将重启.对于内核开发而言,此时重启将不利于我们观察和debug,因此JOS的QEMU经过了特殊定制,此时将打印寄存器和`triple fault`信息.

目前我们可以通过使用gdb来判断我们是否进入了用户空间.通过`make qemu-gdb`命令,使用gdb来调试JOS,并在`env_pop_tf`设下断点.该函数是JOS进入用户空间前,最后运行的一个函数.使用命令`si`来单步调试,CPU在执行指令`iret`后,将进入用户空间.

在用户空间第一条执行的指令应该是`lib/entry.S`中`label start`中的第一条指令`cmpl $USTACKTOP, %esp`.现在通过命令`b *0x...`在`hello`程序的`sys_cputs()`函数中的`int $0x30`处设下断点.具体函数地址请参考`obj/user/hello.asm`.这个`int`指令是请求内核在console上显示字符.如果你无法执行到这里,说明之前的实现有问题,请返回检查并修正.


---

## 处理中断和异常
目前用户空间执行`int $0x30`是死路一条:如果CPU进入用户空间,目前还没有返回的办法.接下来,我们将要实现基本的异常和系统调用处理,这样内核才能从用户空间程序手中重新获得CPU的控制权.

首先,你需要熟悉x86的中断和异常机制.

### Exercise 3
阅读[80386 Programmer's Manual](https://pdos.csail.mit.edu/6.828/2017/readings/i386/c09.htm) 第九章.
或者[IA-32 Developer's Manual](https://pdos.csail.mit.edu/6.828/2017/readings/ia32/IA32-3A.pdf) 第五章.

在本Lab中,我们使用中断,异常这些术语,它们的定义遵循了Intel的规定.但是在操作系统中,异常,陷阱,中断,错误,终止等并没有特定的含义,所以如果你在其他地方看到这些术语完全可能有不同的含义.

## 受保护的控制转移
异常和中断都是一种受保护的控制转移,都会引起CPU从用户模式切换到内核模式.在这个过程中用户空间的代码不会有任何机会接触到内核函数.在Intel的术语中,`interrupt`是指由外部异步引起的控制转移,比如外部IO设备的通知等.`exception`则是由当前运行的代码引起的同步控制转移,比如除0异常或者访问非法地址等.

为了保证控制转移的确处于受保护的状态,当异常中断发生时,CPU当前运行的代码完全无法影响内核接下来的处理逻辑.即中断异常的处理完全是在内核模式下提前设置好的.x86使用了下面两个机制来保证了控制转移一定是被保护的.


### 中断向量表
内核预先定义好了特定的入口,当中断和异常发生时,CPU将执行这些预先定义好的指令.

x86一共允许设置256个特定入口,每个入口都有一个不同的中断向量号.中断向量号是0~255的整数.通过中断向量号,我们可以知道中断的来源:
* 不同的设备
* 错误
* 用户程序请求
* ...

CPU使用中断向量号作为`interrupt descriptor table(IDT)`中的索引.CPU将会从表中载入如下信息:
* 中断处理程序的内存地址,该地址将被载入EIP寄存器.
* 中断处理程序的code segment,其中包含了中断处理程序将要运行的特权等级.在JOS中,所有的中断处理都处于内核模式,即特权级为0.

### Task State Segment(TSS)
在处理中断前,CPU需要保存当前的寄存器值,比如当前进程的CS,EIP寄存器的值.中断处理完成后,通过之前保存的状态信息恢复运行.当然当前进程的上下文信息必须必须处于受保护的状态,以避免其他的用户进程破坏或者窃取数据.

为了达到这个目的,当x86的CPU因中断从用户态进入内核态时,不光会切换运行特权级,同样会将函数栈切换到内核内存中.TSS就保存了这个内核栈的SS和ESP值.中断处理时,CPU将当前进程的SS,ESP,EFLAGS,CS,EIP,错误号(可选)压入内核栈,然后从中断向量表中载入CS和EIP,再从TSS中载入ESP和SS.前面这些工作都是硬件直接完成的,在我们的JOS和xv6的代码中,我们是看不到相关的软件处理过程的.




















---

