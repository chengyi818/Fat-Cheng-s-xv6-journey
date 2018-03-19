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

* objdump -h obj/kern/kernel

该命令可以查看我们kernel ELF中所有段的信息.

信息包含了段的名字,大小,载入地址,执行地址等等.其中VMA(link address)表示该段执行的内存地址,LMA(load address)表示该段载入的内存地址.

当然现在还有一种称为position-independent的ELF,该文件并没有包含绝对的位置信息.它主要是为了现代的共享库设计的,不可避免地也会带来性能和复杂上的代价.在6.828中,我们并没有使用这样的特性.

* objdump -h obj/boot/boot.out

通常而言,链接地址和载入地址是一致的.我们看到Bootloader中的.text就是这样的,载入地址和链接地址均为0x7c00.

* objdump -x obj/kern/kernel

Bootloader使用ELF的头部信息来决定载入哪些program段,载入到哪里.

```
Program Header:
    LOAD off    0x00001000 vaddr 0xf0100000 paddr 0x00100000 align 2**12
         filesz 0x00007120 memsz 0x00007120 flags r-x
    LOAD off    0x00009000 vaddr 0xf0108000 paddr 0x00108000 align 2**12
         filesz 0x0000a300 memsz 0x0000a944 flags rw-
   STACK off    0x00000000 vaddr 0x00000000 paddr 0x00000000 align 2**4
         filesz 0x00000000 memsz 0x00000000 flags rwx
```

通过查看Program Header,我们能够知道该文件的哪些部分将被载入内存.标有*LOAD*的部分将被载入内存,还有其他一些信息包括vaddr(虚拟内存地址),paddr(物理内存地址),filesz(文件大小),memsz(占用内存大小).

回到`boot/main.c`中,`ph->p_pa`表示了每个程序段的目标物理地址.

## Exercise 5
还记得BootLoader的VMA和LMA么?它们都是`0x7c00`.BIOS正是通过这个信息,才将它们载入到正确的位置并且从正确的位置执行.Bootloader的位置信息,我们是通过`boot/Makefrag`中的`-Ttext 0x7c00`设置的.

下面是一个练习,我们再次使用GDB调试BootLoader,看看哪些指令必须要求Bootloader载入在正确的位置.尝试修改Bootloader的载入位置,看看会发生什么?


### 个人答案
我将0x7c00修改位0x6c00,发现无法运行

## Kernel载入地址和执行地址
在*Exercise 3*中,我们发现内核执行的第一条指令的地址是`0x10000c`,而内核的载入地址和执行地址均为`0x100000`.前面我们不是提到link address就是ELF的执行地址么?

这里还有加入一个新的字段,在ELF文件的头部信息中还有一个字段,称为`e_entry`.它表示该ELF的执行入口.

我们可以通过命令`objdump -f obj/kern/kernel`来查看该字段.对比Bootloader的执行入口,我们发现果然如此.

## Exercise 6
我们可以通过GDB的`x/Nx ADDR`命令,来查看内存中的内容.

我们可以查看BIOS准备跳转到Bootloader的节点,0x00100000后16个字节的内容.再对比Bootloader准备跳转到Kernel的节点,0x7d6b后16个字节的内容.看看有什么区别?

### 个人答案
我用了`x/2x 0x100000`和`x/2x 0x7d6b`来查看内存的内容,好像全是0.







---