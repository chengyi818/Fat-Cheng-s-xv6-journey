# LEC20 虚拟机

# 什么是虚拟机?
1. 模拟一台计算机
2. 作为一个应用程序运行在host主机上
3. 准确
4. 独立隔离
5. 性能优良

# 虚拟机的用途
1. 在一台host主机上,运行不同的操作系统
2. 管理大型主机(以OS为粒度,分配CPU和内存)
3. 内核开发环境(比如qemu)
4. 更好地异常错误隔离

# 虚拟机准确性要求
1. 处理不同操作系统内核的奇怪特性
2. 准确复现bugs
3. 处理恶意软件,保证恶意软件不能沙箱逃逸.
4. 目标:
  4.1 运行在虚拟机中的软件无法区分是运行在虚拟机还是真实主机中.
  4.2 运行在虚拟机中的软件无法从虚拟机中逃逸
5. 一些不太好的虚拟机,需要修改guest内核

# 虚拟机的起源
虚拟机是一个很古老的概念.
1. 1960s,IBM使用虚拟机来共享大型计算机.
2. 1990s,VMWare在x86上使得虚拟机重新流行起来.

# 术语
1. VMM: Virtual Machine Monitor,类似host
2. host: 运行虚拟机的主机
3. guest: 运行在主机上的虚拟机程序,包括guest内核及应用程序.
4. 一般虚拟机需要运行在host的操作系统上,host os和vm是独立的.

# VMM的作用
1. 在guest之间分配内存
2. 在guest之间分配CPU时间
3. 为每个guest模拟虚拟硬盘,网络设备等

# 模拟器的缺点
1. VMM解释执行每个guest的指令
2. 保存每个guest虚拟机的状态,比如eflags,cr3等寄存器.
3. 性能比较差

# 改进点: 尽可能直接在CPU上直接运行guest指令
1. 对于大多数指令是可行的.比如`add %eax, %ebx`
2. 问题在于,如何阻止guest执行特权指令.guest执行特权指令很可能会导致VMM和其他guest异常.

# 进一步改进: guest内核运行在非特权级(CPL=3)
1. 普通指令都可以正常执行.
2. 特权指令会陷入VMM中执行
3. VMM将会对guest的虚拟状态,执行特权指令.即**陷入+模拟**

# 陷入+模拟举例---CLI/STI
1. VMM为guest创建了虚拟的IF标志寄存器.
2. VMM控制硬件IF寄存器,可能在guest执行CLI指令后,仍然保持中断开启状态
3. VMM通过虚拟IF寄存器来决定何时中断guest.
4. 当guest执行CLI或STI:
  4.1 因为guest处于CPL=3,因此无法执行该指令,从而导致中断
  4.2 硬件将会调用VMM注册的中断处理函数.
  4.3 VMM查看虚拟CPL状态.若为0,则改变虚拟IF.若不为0,则模拟一个保护异常到guest内核中.
5. VMM必须保证guest只能看到虚拟的IF,且无法接触真实的IF

# 在x86上实现陷入+模拟比较困难
1. 不是所有特权指令在CPL=3下,都会触发中断.
  1.1 `popf`会忽略对中断标志位的修改
  1.2 `pushf`会暴露真实的中断标志位
2. 陷入在x86上性能很差.
3. VMM必须时刻检查PTE的可写标志位,权限不是通过特权指令实现的.

# x86上哪些状态是必须对guest隐藏的?
1. CPL(即CS寄存器低两位),实际为3,而guest期望为0.
2. gdt描述符,实际为3,而guest期望为0.
3. gdtr,指向虚拟的gdt
4. idt描述符,中断会陷入VMM,而不是guest内核.
5. idtr
6. 页表(不指向期望的物理地址)
7. cr3寄存器,指向虚拟的页表
8. EFLAGS中的IF标志位
9. cr0寄存器等.

# VMM如何为guest创建物理内存专用的假象?
1. guest期望起始的物理地址为0,使用所有的物理内存.
2. VMM需要同时支持多个guest,因此guest不可能真使用物理内存0地址.
3. VMM同时需要保护guest的内存,不受其他guest的影响.

## 主要思想
1. 声明的内存大小,小于真实内存大小.
2. 确保分页功能开启.
3. 为guest页表创建一份影子拷贝.
4. 拷贝页表,为guest将虚拟地址指向不同的物理地址.
5. 真实的cr3寄存器,指向拷贝页表
6. 虚拟的cr3寄存器,指向guest页表

## 举例
1. VMM为guest分配物理内存0x1000000 ~ 0x2000000
2. 如果guest修改cr3寄存器,此时guest处于`CPL=3`,VMM将陷入中断处理
3. VMM将guest页表拷贝到影子页表.
4. VMM在影子页表中,为每个物理地址增加0x1000000.
5. VMM检查影子页表,确保所有物理地址均不大于0x2000000.

## 为什么VMM不能直接修改guest页表.

# 影子GDT,IDT
1. 类似内存的处理方式,同样处理GDT和IDT.
2. 真实的IDT指向VMM的中断向量表.
  2.1 VMM如果需要的话,可以跳转到guest内核.
  2.2 VMM可以模拟出磁盘中断.
3. 真实的GDT允许guest内核在CPL=3运行.

# 注意
1. 如果guest尝试修改cr3,gdtr等寄存器,我们依赖于硬件来陷入VMM.
2. 如果guest读取上述寄存器,我们是否需要陷入中断.

# 在CPL=3读取敏感数据是否一定会导致中断
1. `push %cs`得到的CPL=3,而不是0.
2. `sgdt`会暴露真实的GDTR.
3. `pushf`会将真正的IF压栈.
  3.1 假设guest关闭IF,VMM并不会关闭真正的IF,仅仅是关闭guest的虚拟IF.
4. `popf`在CPL=3时,会忽略IF标志位,不会引起中断.所以VMM无法获悉guest内核想要触发中断.
5. `IRET`: 并没有从内核态返回用户态,因此不会恢复`SS/ESP`寄存器.

# 如何处理不会引起中断,暴露真实状态的指令?
1. 修改guest指令,添加`INT 3`,触发中断.
2. 跟踪原始指令,在VMM中模拟执行.
3. `INT 3`只有一个字节,因此不需要修改原代码的大小和布局.
4. 这是论文中的`Binary Translation`的简化实现版本.

# 修改guest指令时,如何获取指令边界?
换言之,修改指令时,如何明确哪些是数据,哪些是指令?VMM可以在符号表中找到函数入口么?

## 主要思想
1. 仅在执行时扫描,因为代码执行时会暴露出指令的边界.
2. 内核的开始部分如下:
`****
  entry:
    pushl %ebp
    ...
    popf
    ...
    jnz x
    ...
    jxx y
  x:
    ...
    jxx z
`****
3. 当VMM首次载入guest内核,修改从入口到第一跳的代码.
  3.1 修改popf为int 3
  3.2 修改jump为int 3
  3.3 然后启动guest内核.
4. 当int 3陷入VMM后
  4.1 确定跳转的地址,此时我们已经知道代码边界.
  4.2 对每个跳转,修改入口到下一次jump的代码.
  4.3 然后跳转到入口点执行.
5. 记录修改的内容,从而避免二次修改.

# 间接调用/跳转
1. 和前面类似,鉴于每次跳转的地址可能不同,所以无法将原始jump替换为int3.
2. 所以,每次必须陷入VMM中断处理

# 函数返回
1. 等同于通过栈上保存的地址间接跳转.
2. 因为无法确定保存的地址是否一定是函数调用,所以每次必须陷入VMM中断处理.

# guest读写自己的代码
1. guest不能看到VMM添加的int3指令
2. 必须再次修改guest修改的代码
3. 是否可以通过页保护机制来**陷入+模拟**读写?
  3.1 不可以,无法在没有读权限的情况下,设置PTE的执行权限.
4. 或许可以令`CS != DS`
  4.1 将修改的代码保存在CS寄存器
  4.2 将原始代码保存在DS寄存器
  4.3 为原始代码设置写保护
5. 当guest尝试修改代码,将会陷入写中断
  5.1 VMM模拟写操作
  5.2 VMM根据guest修改的内容,再次修改guest指令
  5.3 在覆盖修改的指令中,必须确认第一条指令的边界.

# 是否需要再次修改guest的用户态代码
1. 理论上是需要的,比如SGDT, IF等.
2. 但在实践中是不需要的.因为用户态代码只会调用INT,从而陷入VMM.

# 如何处理页表?
1. VMM创建了影子页表,从而映射了不同的物理内存.
2. 扫描每个cr3寄存器指向的页表,从而创建影子页表.

# 如何处理在进程上下文切换过程中guest频繁修改cr3?
1. 主要思想: 延迟拷贝影子页表.
2. 在初始时,仅创建一个空的影子页表.
3. 在guest载入cr3,开始运行后一定会触发很多缺页异常,在此时再根据需要将guest页表的内容拷贝到影子页表.

# 如何处理guest频繁在一组页表间切换?
1. 比如在一组运行的进程间,不断切换.
2. 在这个过程中,并不会修改页表内容,因此反复扫描(包括延迟复制)都是浪费的.
3. 主要思想: VMM可以缓存多个影子页表.
  3.1 cache通过guest页表的地址来索引
4. 通过将guest页表缓存,guest进程间切换速度会快得多.

# guest内核尝试修改页表
1. 保存指令是非特权指令,因此不会触发中断.
2. VMM需要知道这次修改么?
  2.1 如果VMM缓存了多份guest页表拷贝,是的.
3. 主要思想: VMM为guest的PTE设置写保护
  3.1 通过写保护机制触发VMM中断处理,从而模拟,进一步修改影子页表.

# 论文主要讨论了三个方面的妥协和折中
1. 跟踪消耗/隐藏缺页异常/上下文切换消耗
2. 减少其中一个的消耗,会增加另外两个的消耗.
3. 而三个方面都非常重要.

# 如何保护guest内核免于guest用户态的破坏?
1. 均处于CPL=3的用户特权级
2. 可以在`IRET`后将guest内核的PTE清除,在`INT`指令后,将guest内核的PTE重新映射.

# 如何处理硬件设备?
1. `INB`和`OUTB`指令将会触发VMM中断处理.
2. DMA地址是物理内存地址,因此VMM必须转换和检查.
3. guest通常使用真实物理设备没有什么实际意义
  3.1 guest通常需要和其他guest共享物理设备
  3.2 每个guest使用物理磁盘的一部分
  3.3 每个guest看起来都是网络上的一个独立主机
  3.4 每个guest都有一个x-window界面
4. VMM可能会虚拟部分网络和磁盘设备
5. 另外guest也可能执行特殊的驱动,以跳转到VMM.

---

# 论文相关

# 两个重要问题
1. 如何处理会暴露特权信息的指令?比如,`pushf`会暴露`CS`寄存器的低位.
2. 如何尽可能减少消耗较大的中断陷入?

# VMware的答案: 二进制转换(binary translation, BT**
用做正确的代码替换有问题的指令,代码必须能够访问该guest的VMM虚拟状态.

# BT示例
1. `CLI/STI/pushf/popf**,读写虚拟IF.
2. 检测修改PTE的指令
  2.1 设置PTE相关内存写保护,第一次写时会触发中断,然后VMM改写指令.
  2.2 根据真实的页表,VMM会修改相应的影子页表.

# 如何对guest隐藏VMM状态?
1. 非特权的BT指令现在会读写VMM状态
2. 将VMM状态放在非常高的内存地址
3. 通过段寄存器来限制guest使用最后几个内存页.
4. 但是通过设置`%gs`寄存器来允许BT指令读写这些页.

# BT的挑战
1. 难以确定指令边界,难以区分指令和数据.
2. 转换后的指令大小不同
  2.1 导致代码中的指针地址会发生变化
  2.2 程序期望看到原始的fn指针地址,而返回的是修改后栈上的地址.
  2.3 转换后的指令在使用前必须映射
  2.4 因此,每个`RET`指令都必须查找VMM状态信息.

# Intel/AMD对虚拟机的硬件支持
1. 通过硬件支持,实现一个性能尚可的VMM,变得容易得多.
2. 硬件直接保存每个guest的虚拟状态,包括CS,EFLAGS,idtr等.
3. 硬件能够获悉正运行在**guest**模式
  3.1 指令可以直接修改guest虚拟状态,从而避免了过多的陷入到VMM.
4. 硬件增加了一个新的运行级
  4.1 VMM模式,有CPL=0和CPL=3
  4.2 guest模式,CPL=0是一个不完全的特权级.
5. 系统调用时不会陷入VMM,硬件会处理CPL转换.
6. 内存和页表处理?
  6.1 硬件支持两个页表,guest页表和VMM页表.
  6.2 guest的内存引用需要两次查找,guest页表中的物理内存转换需要通过VMM页表.
  6.3 因此guest可以直接修改其自身的页表,而无须VMM创建影子页表.
    6.3.1 不需要VMM为guest页表设置写保护
    6.3.2 不需要VMM跟踪cr3寄存器的变化
  6.4 同时,VMM可以保证guest只会访问其自身的内存.
    6.4.1 只会在VMM页表中映射guest内存.
