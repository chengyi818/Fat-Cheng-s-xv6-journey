# Part2 虚拟内存

## Exercise 2 
熟悉x86的保护模式架构,主要分为分段和分页两种机制.

阅读[Intel 80386 Reference Programmer's Manual](https://pdos.csail.mit.edu/6.828/2017/readings/i386/toc.htm)中的第5和第6章.尤其是*5.2 Page Translation*和*6.4 Page-Level Protection*两小节.

## 虚拟地址, 线性地址, 物理地址
x86中的*虚拟地址*由段地址和段内偏移两部分组成.

*线性地址*是进行了段转换而没有进行页转换前的地址.

*物理地址*是最终进行了段转换和页转换后的地址,也是最终在硬件总线上对内存寻址的地址.

示意图如下:

```

           Selector  +--------------+         +-----------+
          ---------->|              |         |           |
                     | Segmentation |         |  Paging   |
Software             |              |-------->|           |---------->  RAM
            Offset   |  Mechanism   |         | Mechanism |
          ---------->|              |         |           |
                     +--------------+         +-----------+
            Virtual                   Linear                Physical
```

---

C语言中的指针代表的是虚拟地址中的段内偏移.在`boot/boot.S`中,我们通过`lgdt gdtdesc`指令载入了Global Descriptor Table (GDT).仔细观察`gdtdesc`的定义,我们会发现所有段的起始地址均为0,而限制均为0xffffffff,因此此时段地址是不起作用的,也就是说线性地址此时和虚拟地址相等.

在Lab3中,我们将会涉及到通过GDT来设置特权等级.在Lab2中,我们将会聚焦于页表转换,也就是从线性地址转换为物理地址的过程.

在Lab1 Part3`kern/entry.S`中,我们设置了一个简单的页表`entry_pgdir`,将物理内存0~4MB映射到虚拟地址`0xf0000000~0xf0400000`.在JOS下面的课程中,我们将映射256MB的物理内存,虚拟内存起始地址为`0xf0000000`.


## Exercise 3
通过`make qemu`来启动qemu调试jos,然后按`ctrl-a c`可以进入qemu自带的调试器.

使用GDB的xp和x命令来查看内存,确保自己能通过物理内存地址和虚拟内存地址看到相同的内容.

使用info pg和info mem查看虚拟内存和页表的情况.

本练习的主要目的就是熟悉页表的映射关系,可以读懂`info pg`打印的页表信息.

---

在`boot/boot.S`中,我们将CPU切换到了保护模式.从那以后所有代码不再能够直接使用线性地址和虚拟地址,所有的地址均被解释为虚拟地址,并且都需要通过MMU转换.正因为此,所有的C指针也都是虚拟地址.

JOS的代码中经常需要操作地址,有时是物理地址,有时是虚拟地址.为了区别它们,我们用`uintptr_t`表示虚拟地址,用`phyaddr_t`表示物理地址.当然它们其实都是`uint32_t`.

在JOS中,我们可以将一个`uintptr_t`转化为一个指针,并直接使用.但是我们不能将一个`phyaddr_t`转化为一个指针直接使用,因为MMU会将所有的地址都当做虚拟地址处理.

总结:

| C type | Address type |
| --- | --- |
| T*  |	Virtual |
| uintptr_t | 	Virtual|
| physaddr_t | 	Physical|

Q&A:
下面x是哪种类型,`uintptr_t`还是`physaddr_t`?
```
	mystery_t x;
	char* value = return_a_pointer();
	*value = 10;
	x = (mystery_t) value;
```

JOS有时在操作内存时,必须要知道内存的物理地址.比如在进行页表设置时,就需要分配物理内存,并初始化.然而内存并不能越过MMU,因此内核不能直接操作物理内存.

为了操作物理内存,JOS采取的方法是将物理地址0开始的内存,线性地映射到了虚拟地址0xf000000.
因此当我们想要操作物理地址x时,只需要操作将物理地址加上0xf000000从而得到虚拟地址,通过宏KADDR(x)可以简化这个操作.

通过地,如果我们想要通过虚拟地址得到物理地址,可以通过宏PADDR(va)来进行.可以想到这个宏就是将虚拟地址减去了`0xf0000000`.

---

## 引用计数
在后面的课程中,我们将会遇到将同一个物理地址映射到多个虚拟地址的情况,因此我们需要记录物理内存页被引用的次数.

在jos中我们将这个计数保存在`struct PageInfo`中的`pp_ref`字段中.当引用计数为0时可以将该物理页还给内核,因为不再有人使用这个物理页了.换句话说,一个物理页的引用计数应当等于这个物理页在所有页表(page table)中对应的虚拟地址在`UTOP`之下的出现次数之和.高于`UTOP`的虚拟地址属于内核空间,不应该被释放,因此也没有维护引用计数的必要.

另外要注意的是关于`page_alloc`的使用,该函数仅仅是返回了一个物理页管理结构体`sturct PageInfo*`,并没有将与之关联的物理页引用计数加一,因此引用计数的维护应当由调用者完成.比如在`page_insert`中,会将物理页引用计数加一.

## Excecise 4
完成一系列管理页表的函数:插入页表映射关系,移除页表映射关系,创建页表等:
```
        pgdir_walk()
        boot_map_region()
        page_lookup()
        page_remove()
        page_insert()
```
`mem_init()`中调用的`check_page()`,将会检查所有的页表管理函数.请确保通过检查.

---













---