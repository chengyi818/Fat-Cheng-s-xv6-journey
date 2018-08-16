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
	






















































---

