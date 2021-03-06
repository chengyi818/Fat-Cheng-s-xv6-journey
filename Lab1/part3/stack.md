# 栈
在最后我们将探讨x86上的C语言栈.同时,我们要实现一个内核调试函数: 打印函数的调用轨迹(backtrace).

## Exercise 9
1. 确定内核在初始化栈后,栈的起始位置?
2. 内核是如何保证栈的空间的?
3. 初始化栈之后,栈又在何处结束?

### 成胖子个人解答
在`kernel.asm`中,在调用C代码之前两行,对esp和ebp进行了赋值.
```
f010002f:	bd 00 00 00 00       	mov    $0x0,%ebp
f0100034:	bc 00 00 11 f0       	mov    $0xf0110000,%esp
```
紧接着将跳转到函数`i386_init`.在函数`i386_init`中,首先将ebp入栈,其次将esp赋给ebp,接着让esp向下移动.

回到问题中.
1. 栈起始位置是静态写死的,为虚拟地址`0xf0110000`.考虑到映射关系物理地址为`0x00110000`.
2. 栈空间最多为64k,因为虚拟地址`0xf0100000`为BIOS占用的内存,不能覆盖这个部分.我没有看到代码部分,但是栈访问1MB以下的物理空间应该会panic.


---
x86的栈指针寄存器(esp)指向当前栈的最低地址,该地址以下的栈空间都是free的.当值入栈时,esp先向下移动,再将值写入esp指向的位置.当值出栈时,首先读取esp指向的值,然后将esp向上移动.在32位模式,栈中只能保存32bit数值,且esp总能被4整除.一些x86指令,比如call,是直接使用esp寄存器中的值的.

栈底寄存器(ebp)指向栈的底部.当进入一个C语言函数时,首先将前一个函数的ebp寄存器值入栈,然后当前的esp值赋给ebp.如果我们所有的函数调用都遵循这样的规则,那么在任意函数中,我们都可以回溯出函数的调用路径.当我们的程序出问题的时候,这样的回溯会对调试有非常大的帮助.

## Exercise 10
为了加深我们对C语言函数调用规则的了解,在`obj/kern/kernel.asm`中找到函数`test_backtrace`并设下断点.看看它每次被调用时发生了什么?每次被调用时,入栈了哪些参数?

### 成胖子个人解答
这里我们主要需要学习x86的函数调用过程.网上有很多相关的博客.这里给出两个:
1. 精简: [X86架构上函数调用过程的堆栈](https://blog.csdn.net/do2jiang/article/details/5404816)
2. 详细: [函数调用过程探究](https://www.cnblogs.com/bangerlee/archive/2012/05/22/2508772.html)
对于我们而言,主要需要学习的是ebp寄存器的妙用.

---

上面的练习应该让你对我们要实现的stack backtrace函数有了些感觉.在`kern/monitor.c`中,6.828已经准备好了函数原型`mon_backtrace()`.`inc/x86.h`中的函数`read_ebp()`会对我们实现`mon_backtrace()`有所帮助.当我们实现之后,应该将`mon_backtrace`加入`kern/monitor.c`中的commands中,以便从命令行调用.

输出格式要求如下:
```
Stack backtrace:
  ebp f0109e58  eip f0100a62  args 00000001 f0109e80 f0109e98 f0100ed2 00000031
  ebp f0109ed8  eip f01000d6  args 00000000 00000000 f0100058 f0109f28 00000061
  ...
```

每行包含`ebp`, `eip`和`args`.其中`ebp`表示当前函数使用的的栈底指针,也是刚进函数时栈顶指针的位置.`eip`表示函数返回时的返回指令指针,即函数返回后将要执行的指令位置,通常指向函数调用指令(call指令)的下一条指令.

最后`args`列出了函数使用的前5个参数,在函数调用前,这些参数将被压栈.当然这里只是列出了栈上的5个值,事实上我们无法得知参数的个数.

## 指针复习
这里罗列出了*K&R C*第五章的一些要点,是我们后续课程的一些重点.

* 如果`int *p = (int *)100`,那么`(int)p+1`和`(int)(p+1)`的值是不同的.前一个是101,而后一个是104.当一个指针加上一个整数时,整数会被隐式地乘以指针指向对象的大小.

* `p[i]`和`*(p+i)`是相同的,表示p指针指向的第i个对象.

* `&p[i]`和`(p+i)`相同,表示p指针指向的第i个对象的地址.

C程序一般不需要在整型和指针类型间转换,但是在操作系统中我们常常这么做.当你看到一个内存地址相关的加法时,注意区分这是一个整型加法还是一个指针加法.

## Exercise 11
根据上面的描述,实现`backtrace`函数.


---

此时,我们通过`backtrace`命令已经可以查看函数的调用栈,但是我们还是希望得到更多的信息,比如函数名,函数位置等.为了实现这样的功能,JOS已经准备了一个名为`debuginfo_eip()`的函数,它将在符号表中搜索`eip`,并返回相关的信息.该函数定义在`kern/kdebug.c`.

## Exercise 12
修改`backtrace`函数的实现,以便打印出函数名,文件名,行号等更多的信息.

在`debuginfo_eip`中,`__STAB_*`从何而来?这个问题比较复杂,可以拆解为以下问题:
1. 在`kern/kernel.ld`中查看`__STAB_*`
2. 运行`objdump -h obj/kern/kernel`
3. 运行`objdump -G obj/kern/kernel`
4. 运行`gcc -pipe -nostdinc -O2 -fno-builtin -I. -MD -Wall -Wno-format -DJOS_KERNEL -gstabs -c -S kern/init.c`,然后查看`init.S`.
5. 看看BootLoader是如何在载入内核时,载入符号表的.

通过插入`stab_binsearch`函数的调用,补充完成函数`debuginfo_eip()`.通过调用`debuginfo_eip()`扩展`backtrace`函数的实现,实现如下打印:
```
K> backtrace
Stack backtrace:
  ebp f010ff78  eip f01008ae  args 00000001 f010ff8c 00000000 f0110580 00000000
         kern/monitor.c:143: monitor+106
  ebp f010ffd8  eip f0100193  args 00000000 00001aac 00000660 00000000 00000000
         kern/init.c:49: i386_init+59
  ebp f010fff8  eip f010003d  args 00000000 00000000 0000ffff 10cf9a00 0000ffff
         kern/entry.S:70: <unknown>+0
K> 
```
每行包含了文件名和`eip`指向的行号.紧接着是函数名和`eip`距函数入口的指令偏移,比如`monitor+106`表示返回地址`eip`距离函数入口`monitor`为106字节.

### Tip
`printf`函数提供了一种打印没有终止符("\0")字符串的方法.使用man手册,看看它是如何实现的.
```
printf("%.*s", length, string)
```

有时候,我们会发现`backtrace`打印的内容要少很多,这是因为部分函数被编译优化内联了.如果在`GNUMakefile`86行的位置,调整`-O`选项的数值,可以调整编译器的优化程度.

### FC解答
这部分在考察ELF文件结构中STAB和STABSTR的作用和解析,关键在于从STAB中提取出调试信息.















---