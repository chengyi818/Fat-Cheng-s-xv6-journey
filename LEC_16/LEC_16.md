# LEC16 操作系统架构

# 主题: 内核应该做什么?
1. 内核应该提供哪些系统调用?
2. 内核应该提供哪些抽象?

# 取决于上层应用的需求和程序员的品味
1. 没有一个标准的完美答案.
2. 关于答案,有很多选择和讨论.这方面的论文也很多.
3. 本章节主要介绍这方面的思想,而不是具体的实现机制.

# 传统方法
1. 大的抽象概念
2. 宏内核实现,比如Unix, Linux, xv6.

# 栗子: CPU的传统处理方式
1. 内核赋予每个进程独立的"虚拟"CPU,各进程间互不影响.
2. 影响:
  2.1 为了对进程透明,中断必须保存/恢复所有寄存器.
  2.2 定时器中断强制执行透明地上下文切换
3. 优点: 简单,将许多底层细节抽象化.
4. 缺点: 过度抽象,比如进程调度等.可能导致性能下降.

# 栗子: 内存的传统处理方式
1. 每个进程有自己私有的独立地址空间.
2. 优点:
  2.1 进程不需要关心其他进程如何使用内存.
  2.2 因为私有,所以进程不需要关心安全问题
  2.3 内核有很大的自由空间来通过虚拟内存机制实现各种tricks
3. 缺点: 实现用户空间层面的tricks,变得困难.

# 传统内核巧妙实现的内存技巧
1. 延迟物理内存分配--使得分配大块内存快速完成,同时节约了实际物理内存.
2. `fork`时,写时拷贝,快速创建进程.
3. 内存页面交换:
  3.1 进程要求分配的内存大于实际的物理内存?
  3.2 将暂时不用的内存页写入到磁盘中,同时将PTE标记为invalid.
  3.3 当进程需要访问已经被换出的内存时,MMU将会触发`page fault`.
  3.4 内核执行中断处理程序,将被换出的内存再次载入物理内存,并将PTE标记为valid.
  3.5 最后,返回用户程序执行.整个过程对于进程而言是透明的.
  3.6 这套机制能够生效的原因在于,进程在一段时间内,只会访问一段内存.
4. 在可执行文件和共享函数库之间,共享物理内存.
5. 通过共享内存,在内核和用户空间之间,实现零拷贝IO.

# 传统内核背后的哲学要点: 抽象
1. 可移植的接口
  1.1 文件file抽象,隐藏了磁盘硬件的所有寄存器细节.
  1.2 内存地址空间,隐藏了MMU的细节.
2. 简单的接口,将复杂性隐藏
  2.1 通过文件描述符fd和`read()/write()`来执行IO操作,而不是操作具体的设备.
  2.2 访问地址空间时,对进程而言,磁盘换页是透明的.
3. 抽象帮助内核管理资源
  3.1 通过进程的抽象,内核才可以实现进程调度.
  3.2 通过目录和文件的抽象,内核才可以管理硬件磁盘布局.
4. 抽象可以增强内核的安全性
  4.1 文件访问的权限
  4.2 进程及其独立内存地址空间
5. 很多抽象,比如文件描述符,虚拟地址空间,文件名,进程号等.帮助内核实现了虚拟化,隐藏,撤回,调度等等功能.

# 抽象对于应用程序开发人员来说是一个胜利
应用程序开发人员希望花时间构建新的应用程序功能,他们想让操作系统处理所有其他事情,所以他们想要强大,可移植和足够快的内核.

# 传统内核通常是宏内核
1. 整个内核就是一个大的可执行文件.
2. 对于各子系统而言,互相调用非常方便.因为没有明显的边界.比如分页和文件系统缓存等.
3. 所有内核代码均运行在特权级,并没有内部安全限制.

# 传统内核的缺点
1. 大: 复杂,有缺陷,容易不可靠(原则上,实际上不太可靠)
2. 过于抽象,导致性能下降.比如上下文切换时,并不需要保存全部寄存器.
3. 抽象可能不正确.比如需要等待一个并非子进程的进程执行.
4. 抽象可能妨碍用户程序的优化.数据库可能比内核文件系统更擅长在磁盘上布局`B-Tree`文件.

# 另一个可选的架构: 微内核
1. 主要思想: 将大多数操作系统功能转移到用户空间服务进程
2. 内核可以尽量小,主要是IPC进程间通信的实现.
3. 目标:
  3.1 简单的内核实现应该更快和更可靠
  3.2 服务进程易于替换和修改
4. 实现示例: Mach3, Minix
5. JOS是微内核micro-kernel和外内核exo-kernel的混合.

# 微内核的优点
1. IPC进程间通信会更快
2. 独立的服务迫使内核开发人员考虑模块化
3. 好的IPC非常适合新的用户态服务,例如X server

# 微内核的缺点
1. 内核不会太小: 需要实现进程和内存管理
2. 可能需要很多次IPC,总的来说很慢
3. 很难将内核分成许多服务进程! 这使得跨服务优化进程变得更加困难,所以服务进程往往很大,这不是一个巨大的优点.

# 微内核已经取得了一些成功
1. IPC/Service理念广泛应用于OSX,Android等,但是对于传统的内核服务来说并不多,主要用于(大量)新服务,架构设计为客户端/服务端.
2. 一些嵌入式操作系统,有强烈的微内核风格.

---

# 外内核(1995)

# 论文
1. 操作系统社区投入了巨大的关注
2. 有很多有趣的想法
3. 描述了一个早期的研究原型
4. 1997年,`SOSP`的论文实现了更多的想法

# 外内核概览
1. 哲学理念: 取消所有的抽象,尽量暴露硬件细节,让应用程序按需定制.
2. 外内核不会提供地址空间,管道,文件系统,TCP等抽象.
3. 应用程序需要直接使用外内核暴露的 MMU, 物理内存, 网卡, 时钟中断
4. 外内核的会导致应用程序的可移植性很差,但是对硬件的控制力会增强.
5. 每个App的libOS实现抽象,可能是POSIX的内存地址空间,fork(), 文件系统,TCP等.每个应用程序有自己定制的libOS,也有自己的抽象.
6. 为什么?由于精简和简单,内核可能会更快.由于应用程序可以根据自己的实际情况定制libOS,所有应用程序也可能更快.

# 外内核面临的挑战
1. 外内核将什么资源暴露给libOS?
2. 如果要在用户空间实现写时拷贝fork(),内核需要实现哪些API?
3. libOS可以共享吗?安全性怎么保证?
4. 没有宏内核提供的抽象,我们可以安全地共享lib么?
5. 通过定制libOS,应用程序能够得到足够的好处么?

# 外内核内存接口
## 暴露的资源
内核将会暴露物理内存页和MMU中`VA->PA`之间的映射关系.

## 应用程序到内核的API
```
    pa = AllocPage()
    TLBwr(va, pa)
    Grant(env, pa)  -- to share memory
    DeallocPage(pa)
```

## 内核到应用程序的API
```
  PageFault(va)
  PleaseReleaseMemory()
```

## 外内核需要做什么?
1. 跟踪保存进程和物理内存之间的对应关系.
2. 确保应用程序只为自己拥有的物理内存,创建虚拟内存映射关系.
3. 当系统内存耗尽时,决定要求哪个应用程序放弃物理内存页,这个应用程序可以自行决定放弃它的哪个内存页.


# 虚拟内存相关的典型使用场景
应用程序希望分配100MB的不连续数组,这里延迟分配的实现和之前作业中的实现不同.
```
  PageFault(va):
    if va in range:
      if va in table:
        TLBwr(va, table[va])
      else:
        pa = AllocPage()
        table[va] = pa
        TLBwr(va, pa)
      jump to faulting PC
```

# 通过外内核的内存管理,我们可以完成一些有意思的操作
## 数据库喜欢在内存中缓存磁盘页面.
在传统内核中会遇到一些困难,假设操作系统支持在磁盘上按需分页.
1. 如果数据库缓存了一些数据,而操作系统需要一个物理页面,操作系统可以换出包含缓存磁盘块的数据库页面.
2. 但那会浪费时间: 如果数据库知道缓存数据被换出,它会释放相关物理物理,然后从数据库文件中重新读取.

## 微内核场景
1. 微内核需要从某些应用程序取回物理内存.
2. 微内核向DB进程发送了一个`PleaseReleaseMemory()`上调函数.
3. DB进程挑选了一个暂时不用的物理页面,执行`DeallocPage(pa)`.
4. 或者,DB挑选了一个已被修改过的脏页,将其中的内容保存到DB文件系统,然后再执行`DeallocPage(pa)`.

---

# 外内核CPU接口
## 不再提供透明的进程切换
1. 当内核需要运行app时,内核向上调用app.
2. 当内核需要app停止运行时,内核向上调用app.
3. 这些向上调用都是会调用到app的固定地址,对app不再是透明的.

## 当app正在运行,且时间片耗尽,时钟中断触发时
1. CPU中断app运行,陷入内核.
2. 内核通过`please yield`向上调用,跳转回app
3. app保存当前上下文,即所有寄存器
4. app向下调用`yield()`.

## 当内核再次运行app
1. 内核通过`resume`向上调用,跳转到app
2. app恢复之前保存的上下文.

## 外内核无需保存用户态的寄存器
1. 这会使系统调用/陷阱/上下文切换更快

# 通过外内核的CPU管理,app可以完成一些有趣的操作
## 假设时间片在持有锁时,耗尽
1. 我们可能并不希望app进程在持有锁时,被调度.因为这样可能会导致其他apps阻塞.
2. 那么在app执行`please yield()`的时候,可以先完成临界区的工作,从而释放锁.

# 快速IPC
## 传统内核中的IPC
1. pipe或者socket
2. 抽象为一种消息传递
3. 实际实现为内核中的两块buffer
4. 速度比较慢:
  4.1 `write+read+read+write`,需要8次切换上下文

## 外内核Aegis中的IPC
1. `yield()`可以传入一个目标进程参数
2. 内核向上调用目标进程
3. 几乎相当于直接跳转到目标进程的某条指令处
4. 内核只允许跳转到目标进程的特定地址
5. 内核不使用寄存器,所以可以使用寄存器来保存参数和返回值.
6. 目标进程通过`yield()`返回
7. 更快: 只有4次切换,更少的寄存器保存/恢复,没有阻塞的`read()`.

## 注意
1. IPC只有目标进程中执行.
2. TODO

# 底层性能总结
1. 主要是关于快速系统调用,陷阱和向上调用.
2. 系统调用的速度非常重要！
3. 缓慢的系统调用速度,鼓励复杂的系统调用,不鼓励频繁的调用.
4. trap处理不会保存大部分寄存器.
5. 从内核到app,快速的向上调用.(内核不需要处理上下文恢复)
6. 受保护的IPC调用,只能跳转到指定的地址.
7. 将一些内核的数据结构映射到用户空间(page table, 寄存器保存)

# 主要思想--关于抽象
1. 自定义抽象对性能是一个提升,app需要底层操作来完成自定义抽象
2. 大量内核操作可以在用户空间实现,同样保证安全性和共享
3. 保护机制并不需要要内核实现大的抽象.
4. 可以保护没有内核管理地址空间的进程页面,1997年的论文对文件系统进行了全面的开发.
5. 地址空间抽象可以分解为: 物理页面分配和va->pa映射

# 外内核的结果
1. 目前很少使用
2. 以后很难说...

# 总结预期
1. 外内核是一个研究项目.
2. 研究的成功,将会带来影响力.
  2.1 改变人们的思考方式
  2.2 帮助他们看到新的可能性
  2.3 人们可能会借鉴部分思想
3. 研究成功不代表会有很多用户
  3.1 通常研究很少能够转化为产品,即使研究成果很好

# 外内核的影响
1. 相较于1995年,UNIX对用户空间暴露了更多的底层控制接口.这对于某些应用来说,非常重要.
2. 人们思考很多关于内核扩展性的问题,比如Linux内核支持动态加载内核模块.
3. 库操作系统经常被使用,比如`unikernel`.
