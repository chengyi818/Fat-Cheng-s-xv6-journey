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

