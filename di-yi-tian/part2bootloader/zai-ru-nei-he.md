# Loading the Kernel

## Exercise 4
推荐学习K&R C的5.1,5.5节,主要学习指针方面的知识.这部分内容务必掌握,不然后续的课程学习起来会很吃力.

看完相关材料后,对比[pointers.c](https://pdos.csail.mit.edu/6.828/2017/labs/lab1/pointers.c).确保自己了解每条语句以及其运行结果.

## ELF格式
本节接下来了的内容,我们来重点学习ELF结构.ELF是*Executable and Linkable Format*的缩写.

我们知道在C语言编译时,首先将C代码编译成目标文件(.o),然后链接器会将目标文件链接成一个二进制文件,比如`obj/kern/kernel`.这个文件就是ELF格式的文件.

在[the ELF specification](https://pdos.csail.mit.edu/6.828/2017/readings/elf.pdf)有关于ELF格式的详细介绍.我个人很喜欢的一本书[程序员的自我修养：链接、装载与库](https://item.jd.com/10067200.html?jd_pop=fc6a1113-0a47-4732-994d-6ecdc0304351&abt=3)也有关于ELF的详细介绍.

当然6.828中我们并不需要深入地掌握ELF结构.整个ELF结构还是相当复杂的,大部分复杂的部分都是为了支持共享库的动态加载.这个特性我们暂时用不到.

在6.828中,我们可以将ELF结构看做是一个ELF头部和若干个program段的组合体.ELF头部是固定格式,描述了ELF文件整个的一些信息,也包括了program段的个数.program段则包含了代码或者据,也指明了哟啊加载到内存的目标地址.BootLoader并不会修改program段,它只是加载program段到内存中,并运行.

关于C语言中ELF的格式定义,请参见`inc/elf.h`.而在program段中,我们感兴趣是下面这几个段:
* .text段

程序的可执行指令部分

* .rodata段

只读数据,比如ASCII码的字符串.在6.828中,我们并没有配置硬件去保证该段不被修改.

* .data段

已被初始化的全局数据段.

* .bss段
未被初始化的全局变量,比如全局变量int x.在内存中紧跟着.data段.C语言要求这些未初始化的全局变量默认值均为0,所以对于ELF文件而言并不需要在ELF中记录该段的内容,仅仅只需要记录该段的大小和地址即可.在linker将ELF载入内存时,应当在.bss对应的内存区域初始化为0.

### objdump查看ELF格式


















---