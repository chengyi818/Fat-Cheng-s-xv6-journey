# 内核地址空间

JOS将32位的线性地址空间划为了两个部分.用户空间位于低地址部分,内核空间位于高地址部分.它们之间的分割线由`inc/memlayout.h`中的`ULIM`来定义,在虚拟地址空间顶部刚好为内核保留了256MB的空间.

`inc/memlayout.h`文件很重要,对后续课程的完成有不小的帮助.

```

/*
 * Virtual memory map:                                Permissions
 *                                                    kernel/user
 *
 *    4 Gig -------->  +------------------------------+
 *                     |                              | RW/--
 *                     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 *                     :              .               :
 *                     :              .               :
 *                     :              .               :
 *                     |~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~| RW/--
 *                     |                              | RW/--
 *                     |   Remapped Physical Memory   | RW/--
 *                     |                              | RW/--
 *    KERNBASE, ---->  +------------------------------+ 0xf0000000      --+
 *    KSTACKTOP        |     CPU0's Kernel Stack      | RW/--  KSTKSIZE   |
 *                     | - - - - - - - - - - - - - - -|                   |
 *                     |      Invalid Memory (*)      | --/--  KSTKGAP    |
 *                     +------------------------------+                   |
 *                     |     CPU1's Kernel Stack      | RW/--  KSTKSIZE   |
 *                     | - - - - - - - - - - - - - - -|                 PTSIZE
 *                     |      Invalid Memory (*)      | --/--  KSTKGAP    |
 *                     +------------------------------+                   |
 *                     :              .               :                   |
 *                     :              .               :                   |
 *    MMIOLIM ------>  +------------------------------+ 0xefc00000      --+
 *                     |       Memory-mapped I/O      | RW/--  PTSIZE
 * ULIM, MMIOBASE -->  +------------------------------+ 0xef800000
 *                     |  Cur. Page Table (User R-)   | R-/R-  PTSIZE
 *    UVPT      ---->  +------------------------------+ 0xef400000
 *                     |          RO PAGES            | R-/R-  PTSIZE
 *    UPAGES    ---->  +------------------------------+ 0xef000000
 *                     |           RO ENVS            | R-/R-  PTSIZE
 * UTOP,UENVS ------>  +------------------------------+ 0xeec00000
 * UXSTACKTOP -/       |     User Exception Stack     | RW/RW  PGSIZE
 *                     +------------------------------+ 0xeebff000
 *                     |       Empty Memory (*)       | --/--  PGSIZE
 *    USTACKTOP  --->  +------------------------------+ 0xeebfe000
 *                     |      Normal User Stack       | RW/RW  PGSIZE
 *                     +------------------------------+ 0xeebfd000
 *                     |                              |
 *                     |                              |
 *                     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 *                     .                              .
 *                     .                              .
 *                     .                              .
 *                     |~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~|
 *                     |     Program Data & Heap      |
 *    UTEXT -------->  +------------------------------+ 0x00800000
 *    PFTEMP ------->  |       Empty Memory (*)       |        PTSIZE
 *                     |                              |
 *    UTEMP -------->  +------------------------------+ 0x00400000      --+
 *                     |       Empty Memory (*)       |                   |
 *                     | - - - - - - - - - - - - - - -|                   |
 *                     |  User STAB Data (optional)   |                 PTSIZE
 *    USTABDATA ---->  +------------------------------+ 0x00200000        |
 *                     |       Empty Memory (*)       |                   |
 *    0 ------------>  +------------------------------+                 --+
 *
 * (*) Note: The kernel ensures that "Invalid Memory" is *never* mapped.
 *     "Empty Memory" is normally unmapped, but user programs may map pages
 *     there if desired.  JOS user programs map pages temporarily at UTEMP.
 */
```

---

## 权限和错误隔离
因为内核地址空间和用户地址空间是隔离的,因此我们需要使用page table中的权限控制位来限制用户代码访问内核的地址空间.否则用户代码将会有意或无意地覆盖内核数据,从而造成各种各样的错误.值得注意的一点是,可写权限位`PTE_W`同时影响内核代码和用户代码.

用户代码没有权限访问高于`ULIM`以上的内核,而内核代码可以读写这个部分.对于`[UTOP, ULIM)`之间地址空间,内核代码和用户代码都只有权限读取,但没有权限写入.这个部分的地址空间是用于将内核的特定数据提供给用户空间读取的.最后低于`UTOP`的空间是供用户空间使用的,用户代码将有权限访问这部分内存.

---

## 初始化内核地址空间

现在我们将要设置`UTOP`之上的地址空间,也就是内核地址空间.`inc/memlayout.h`是我们将要使用的内核空间布局.我们将会使用到Lab2之前完成的函数来设置线性地址和物理地址之间的映射关系.

### Exercise 5
完成函数`mem_init()`中`check_page()`之后的代码.完成后,代码应该通过`check_kern_pgdir()`和`check_page_installed_pgdir()`的检查.

### Q&A
* Q2: 下面的表格是page directory的示意图,看懂并且尽量填充.

Entry	| Base Virtual Address	| Points to (logically)
--- | --- | ---
1023 |	? | Page table for top 4MB of phys memory
1022 |	? | ?
. | ? | ?
. | ? | ?
. | ? | ?
2 | 0x00800000 | ?
1 | 0x00400000 | ?
0 | 0x00000000 | [see next question]

* Q3: 内核地址空间和用户地址空间在同一个地址空间中,使用同一个page directory进行地址转换,为什么用户代码不能访问内核地址空间?换言之,是什么机制在保护内核地址空间?

* A3: pte中的低12位是可以用于标记权限的,就是通过其中的`PTE_U`这个标志位来控制的.

* Q4: JOS最大可支持的物理内存是多少?为什么?

* Q5: JOS使用了多少内存用于管理内存?假如我们的物理内存为最大可支持大小,管理结构会发生什么问题?

* Q6: 再次浏览`kern/entry.S`和`kern/entrypgdir.c`.当我们开启分页功能后,EIP此时仍然指向一个低地址(稍高于1MB).我们在何处将EIP转移到了高于`KERNBASE`的地址呢?是什么机制让我们可以在EIP运行于高地之后,仍然可以切换到低地址呢?为什么有必要将EIP从低地址切换到高地址?
