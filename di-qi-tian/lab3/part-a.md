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