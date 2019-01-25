# Part1 物理内存页管理
操作系统必须知道当前哪些物理内存正在被使用,哪些又处于空闲状态.JOS以页为单位管理物理内存,因此JOS可以使用MMU来映射和保护已分配的内存.

首先,我们需要完成物理内存页分配器.它的作用是通过一个`struct PageInfo`链表来管理所有空闲的物理页,每个节点对应一个物理页.这是我们需要最先完成的功能,因为页表管理单元也需要分配物理页来存储页表.

## Exercise 1
完成`kern/pmap.c`中的如下函数:
1. boot_alloc()
2. mem_init()
    直到函数`check_page_free_list()`
3. page_init()
4. page_alloc()
5. page_free()

在完成后,`check_page_free_list()`和`check_page_alloc()`将会测试我们的代码.编译启动JOS并确保检测通过.

在开发过程中,使用`assert()`将会有助于开发.