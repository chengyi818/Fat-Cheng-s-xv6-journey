# lab5: 文件系统,spawn和shell

## 引言
首先在本实践中,我们将要实现库函数`spawn`,用于装载和运行磁盘上的可执行文件.接着我们需要完善内核和函数库以支持shell的运行.以上这些Feature需要文件系统的支持,因此本实践中,将会引入一个简单的可供读写的文件系统.

## 准备工作
我们需要执行以下代码,以将代码切换到lab5.
```
athena% cd ~/6.828/lab
athena% add git
athena% git pull
Already up-to-date.
athena% git checkout -b lab5 origin/lab5
Branch lab5 set up to track remote branch refs/remotes/origin/lab5.
Switched to a new branch "lab5"
athena% git merge lab4
Merge made by recursive.
.....
athena%
```

在完成这部分工作之后,会增加一些新文件.其中重要的罗列如下:
1.fs/fs.c
> Code that mainipulates the file system's on-disk structure.
2. fs/bc.c
> A simple block cache built on top of our user-level page fault handling facility.
3. fs/ide.c
> Minimal PIO-based (non-interrupt-driven) IDE driver code.
4. fs/serv.c
> The file system server that interacts with client environments using file system IPCs.
5. lib/fd.c
> Code that implements the general UNIX-like file descriptor interface.
6. lib/file.c
> The driver for on-disk file type, implemented as a file system IPC client.
7. lib/console.c
> The driver for console input/output file type.
8. lib/spawn.c
> Code skeleton of the spawn library call.


由于lab5的部分功能还未完成,我们需要暂时将`kern/init.c`中的`ENV_CREATE(fs_fs)`注释掉,还要将`lib/exit.c`中的`close_all()`注释掉.

现在lab5应该可以正确运行`pingpong`,`primes`和`forktree`等程序.确保这三个程序可以正常运行,再继续下面的练习.
```
$make run-pingpong
$make run-primes
$make run-forktree
```

---

# 文件系统背景介绍
我们将要实现的文件系统比绝大多数真正的文件系统都要简单,当然也比xv6的文件系统要简单.但JOS的文件系统仍然提供了基本的特性.创建,读取,写入,删除文件,并提供了分层的目录结构.

JOS是一个单用户操作系统,因此文件系统也不会支持UNIX系统一般具有的文件所有者或者权限等特性.另外,xv6的文件系统也不支持软硬链接,时间戳,特殊设备文件等特性.

## On-Disk File System Structure 磁盘上文件系统结构
大多数Unix文件系统将可用的磁盘空间分为两个部分: **inode**和**data**.Unix文件系统为文件系统中每个文件分配一个inode,一个文件的inode保存了文件的关键元数据,比如其`stat`属性和指向其数据block的指针等.data区域被划分为大得多(通常为8kb或者更大)的block,文件系统在其中保存文件数据或者目录元数据.目录entry包含了文件名和文件对应的inode指针;如果文件系统中的多个目录entry指向同一个文件的inode,则称该文件是硬链接的.当然JOS的文件系统并不支持硬链接,因此我们不需要这种抽象,从而简化我们的设计: JOS的文件系统根本不会使用inode,而是将文件(或子目录)的元数据保存在描述该文件的(唯一的)目录entry中.

文件和目录在逻辑上都由一系列数据块组成,这些数据块可能分散在整个磁盘上,就像进程的虚拟地址空间的内存页面可能分散在整个物理内存中一样.文件系统隐藏了磁盘布局的细节,提供了读写文件任意位置的接口.作为创建删除文件的一部分,文件系统包含了对目录的修改.JOS文件系统允许用户直接读取目录的元数据(比如read),这意味着用户可以自己执行目录扫描操作(比如ls),而不必依赖文件系统的一些特殊调用.这种目录扫描方法的缺点,也是大多数现代UNIX变体不支持它的原因,是它使应用程序依赖于目录元数据的格式,使得在不改变或至少重新编译应用程序的情况下,很难改变文件系统的内部布局.即用户应用程序和文件系统深度耦合.

### Sectors and Blocks 扇区和块
大多数磁盘不能以字节粒度执行读写,而是以扇区为单位执行读写.在JOS中,扇区大小为512字节.文件系统实际上以block块为单位分配和使用磁盘存储.小心区分这两个术语:扇区大小是磁盘硬件的属性,而block块大小是操作系统使用磁盘的大小.文件系统的block块大小必须是基础磁盘扇区大小的倍数。

UNIX xv6文件系统使用512字节的块大小,与底层磁盘的扇区大小相同.然而,大多数现代文件系统使用更大的block块大小,因为存储空间变得更便宜,并且以更大的粒度管理存储更加有效.JOS的文件系统将使用4096字节的block块大小,和处理器的内存页面大小一致.

### Superblocks 超级块
文件系统通常将某些磁盘块保留在磁盘上"易于查找"的位置(例如最开始或最结尾),以保存描述文件系统整体属性的元数据,例如块大小,磁盘大小,查找根目录所需的任何元数据,文件系统上次装载的时间,文件系统上次检查错误的时间等.这些特殊的块被称为超级块.

我们的文件系统将只有一个超级块,它将始终位于磁盘上的block1.它的布局在`struct Super(inc/fs.h)`中定义.block0通常保留用于保存引导加载器和分区表,因此文件系统通常不使用第一个磁盘块.许多"真实的"文件系统维护多个超级块,这些超级块在磁盘的几个间隔很宽的区域中复制,因此,如果其中一个超级块损坏或磁盘在该区域中出现介质错误,其他超级块仍然可以找到并用于访问文件系统.

![disk](../images/disk.png)

### File Meta-data 文件元数据
描述文件系统中文件的元数据的布局由`inc/fs.h`中的`struct File`描述.该元数据包括文件的名称,大小,类型(常规文件或目录),以及指向组成文件的块的指针.如上所述,我们没有索引节点,因此元数据存储在磁盘的目录条目中.与大多数“真实的”文件系统不同,为了简单起见,我们将使用`struct File`来表示文件元数据,它在磁盘和内存中是一致的.

`struct File`中的`f_direct`数组包含存储文件的前10个块的块号的空间,我们称之为文件的直接块.对于大小高达10*4096 = 40KB的小文件,这意味着文件所有块的块号将可以直接保存在`struct File`本身.然而,对于较大的文件,我们需要一个存放文件其余块号的地方.因此,对于任何大于40KB的文件,我们会分配一个额外的磁盘块,称为文件的间接块,以容纳最多4096/4 = 1024个额外的块号.因此,我们的文件系统允许文件的大小达到1034个块,或者仅仅超过4兆字节.为了支持更大的文件."真实"文件系统通常也支持双间接和三间接块.

![file](../images/file.png)

### Directories versus Regular Files 目录和普通文件
我们文件系统中的`struct File`可以表示普通文件或目录;这两种类型的"文件"通过文件结构中的`type`字段来区分.文件系统以完全相同的方式管理普通文件和目录文件,除了它根本不解释与普通文件相关联的数据块的内容,而文件系统将目录文件的内容解释为描述目录内的文件和子目录的一系列`struct File`.

我们文件系统中的超级块包含一个`struct File`(`struct super`中的`root`字段),它保存文件系统根目录的元数据.该目录文件的内容是描述文件系统根目录中的文件和目录的`sturct File`序列.根目录中的任何子目录都可能包含更多代表子目录的`struct File`,依此类推.

---

# The File System 文件系统
本实验的目标不是让我们实现整个文件系统,而是让我们实现某些关键组件.特别是,我们将完成将block块读入block缓存并将它们刷新回磁盘;分配磁盘block块;将文件偏移量映射到磁盘block块;并在IPC接口中实现read,write,open.因为我们不会自己实现整个文件系统,所以熟悉所提供的代码和各种文件系统接口非常重要.

## Disk Access 磁盘访问
JOS的文件系统需要能够访问磁盘,但是我们还没有在JOS内核中实现任何磁盘访问功能.我们不采用传统的"宏内核"操作系统策略,即在内核中添加IDE磁盘驱动程序以及必要的系统调用来允许文件系统访问它,而是将IDE磁盘驱动程序作为用户级文件系统的一部分来实现.我们仍然需要稍微修改JOS内核,以便进行设置,使文件系统进程拥有磁盘访问所需的特权.

用户空间中依赖于轮询,可以实现"基于编程输入/输出"(PIO)的磁盘访问,并且不需要使用磁盘中断.也可以在用户模式下实现中断驱动的设备驱动程序(例如L3和L4内核),但是这更加困难,因为内核必须对设备中断进行处理,并将它们分派到正确的用户空间进程中.

x86处理器使用EFLAGS寄存器中的IOPL位来确定是否允许保护模式下的代码执行特殊的设备IO指令,如IN和OUT指令.由于我们需要访问的所有IDE磁盘寄存器都位于x86的IO空间,而不是内存映射,因此,为文件系统环境提供"IO权限"是我们允许文件系统访问这些寄存器所需要做的唯一事情.实际上,EFLAGS寄存器中的IOPL位为内核提供了一种简单的"全有或全无"方法来控制用户空间代码是否可以访问IO空间.在我们的例子中,我们希望文件系统环境访问IO空间,但是我们根本不希望任何其他进程能够访问IO空间.

### Exercise 1
`i386_init`通过将`ENV_TYPE_FS`类型传递给进程创建函数`env_create`来标识文件系统进程.在`env.c`中修改`env_create`,以便它赋予文件系统进程IO权限,但不要赋予任何其他进程该权限.

确保我们可以启动文件系统进程,而不会导致`General Protection`错误.执行`make grade`,我们应该可以通过`fs i/o`测试.

### Question
1. 当我们随后从一个进程切换到另一个进程时,我们是否必须做特殊操作来确保此IO权限设置得到正确保存和恢复?为什么?
A: 不需要,在进程切换时,EFLAGS寄存器会被正确地保存和恢复.

### Tips
请注意,本实验中的GNUmakefile文件将QEMU设置为像以前一样使用文件`obj/kern/kernel.img`作为disk 0的映像(通常是DOS/Windows中"驱动器C"),并将(新的)文件`obj/fs/fs.img`作为disk 1的映像("驱动器D").在本实验中,我们的文件系统应该只接触disk 1;disk 0仅用于引导内核.如果我们不小心损坏了其中一个磁盘映像,我们可以通过键入以下命令将它们重置为原始的"原始"版本:

```
$make clean
$make
```

## The Block Cache 块缓存
在JOS文件系统中,我们将在虚拟内存系统的帮助下实现一个简单的"缓冲区缓存"(实际上只是块缓存).块缓存的代码在`fs/bc.c`中.

JOS文件系统将支持处理3GB或更小的磁盘.文件系统进程地址空间固定为3GB区域,从`0x10000000(DISKMAP)`到`0xd000000(DISK MAP+DISK MAX)`,作为磁盘的"内存映射".例如,磁盘block 0映射到虚拟地址`0x1000000`,磁盘block 1映射到虚拟地址`0x10001000`,依此类推.`fs/bc.c`中的`diskaddr()`函数实现了从磁盘block块号到虚拟地址的转换(以及一些健全性检查).

由于JOS文件系统环境具有独立于系统中所有其他进程虚拟地址空间的虚拟地址空间,并且文件系统进程唯一需要做的事情是实现文件访问,因此文件系统进程的大部分地址空间用于映射磁盘是合理的.在32位机器上真正实现这样的文件系统是很难的,因为现代磁盘通常远大于3GB.在具有64位地址空间的机器上,这种缓冲区高速缓存管理方法仍然是可能的.

当然,将整个磁盘读入内存需要很长时间,因此我们将实现一种按需分页,其中我们只分配磁盘映射区域中的页面,并根据该区域中的`page fault`来从磁盘读取相应的block块.这样,我们可以假装整个磁盘都在内存中.

### Exercise 2
在`fs/bc.c`中实现`bc_pgfault()`和`flush_block()`函数.`bc_pgfault()`是一个`page fault`处理程序,就像我们在前面的实验中为写时复制fork编写的一样,只是它的工作是响应`page fault`并从磁盘加载页面.完善代码时,请记住:
1. `addr`可能不与block块边界对齐.
2. `ide_read`是以sector扇区为单位操作,而不是以block块为单位.

当有需要时,`flush_block()`函数应该将block块写回到磁盘上.如果block块不在block块缓存中(也就是说,页面没有被映射),或者如果它没有被写入,`flush_block()`不应该做任何事情.我们将使用硬件特性来跟踪内存block块自上次从磁盘读取或写入磁盘以来是否已被修改.要查看一个block块是否需要写回磁盘,我们只需查看在uvpt条目中是否设置了`PTE_D`dirty位.(处理器响应对内存页的写入而设置`PTE_D`位;参见386参考手册[第5章](https://pdos.csail.mit.edu/6.828/2011/readings/i386/s05_02.htm)中的5.2.4.3.).将block块写回磁盘后,`flush_block()`应使用`sys_page_map()`清除`PTE_D`位.

使用`make grade`来测试我们的代码.这会儿,我们应该可以通过`check_bc`,`check_super`和`check_bitmap`.

### Tips
`fs/fs.c`中的`fs_init()`函数是如何使用block块缓存的一个主要示例.初始化block块缓存后,它只需将`super`全局变量指向磁盘映射内存.之后,我们可以直接地从`super`结构中读取,就像它们在内存中一样,并且我们的`page fault`处理程序会根据需要从磁盘中读取它们.

## The Block Bitmap
`fs_init()`设置bitmap指针后,我们可以将bitmap视为一个bit位数组,每个bit代表磁盘上的一个block块.例如,请参见`block_is_free()`,它只是检查bitmap中给定的block块是否标记为`free`.

### Exercise3
使用`free_block`作为在文件系统中实现`alloc_block()`函数的参考,它应该在bitmap中找到一个空闲的磁盘block块,将它标记为已使用,并返回该块的编号.分配block块时,应该立即用`flush_block()`将更改后的bitmap block块刷新回磁盘,以保持文件系统的一致性.

使用make grade测试我们的代码.我们的代码现在应该通过"alloc_block".

## File Operations
JOS已经提供了许多函数来实现文件系统所需的基本功能,这些函数包括解释和管理`struct File`,扫描和管理目录文件,以及从根目录开始遍历文件系统以解析绝对路径名.通读`fs/fs.c`中的所有代码,并确保在继续操作之前理解每个函数的功能.

### Exercise4
实现`file_block_walk()`和`file_get_block()`.`file_block_walk()`从文件中的block块偏移量映射到`struct File`或间接块中该块的指针,非常像`pgdir_walk()`对页表所做的操作.`file_get_block()`更进一步,映射到实际的磁盘块,必要时可以分配一个新的磁盘block块.

使用`make grade`测试我们的代码.此时代码应该通过`file_open`, `file_get_block`, `file_flush/file_truncated/file rewrite`和`testfile`.

### Tips
`file_block_walk()`和`file_get_block()`是文件系统的基础工具.例如,`file_read()`和`file_write()`只不过是在操作连续缓冲区,而`file_get_block()`却需要处理分散的block块和顺序缓冲区之间的关系.

## The file system interface 文件系统接口
现在,JOS文件系统进程本身已经拥有了必要的函数,我们必须让其他进程可以使用这些函数.由于进程隔离的机制,其他进程不能直接调用文件系统进程中的函数,因此我们将通过一个远程调用(RPC)接口来封装对文件系统进程的访问,这个抽象是建立在JOS的IPC机制之上的.下图展示了对文件系统进程的访问过程(比如说,read):
```
      Regular env           FS env
   +---------------+   +---------------+
   |      read     |   |   file_read   |
   |   (lib/fd.c)  |   |   (fs/fs.c)   |
...|.......|.......|...|.......^.......|...............
   |       v       |   |       |       | RPC mechanism
   |  devfile_read |   |  serve_read   |
   |  (lib/file.c) |   |  (fs/serv.c)  |
   |       |       |   |       ^       |
   |       v       |   |       |       |
   |     fsipc     |   |     serve     |
   |  (lib/file.c) |   |  (fs/serv.c)  |
   |       |       |   |       ^       |
   |       v       |   |       |       |
   |   ipc_send    |   |   ipc_recv    |
   |       |       |   |       ^       |
   +-------|-------+   +-------|-------+
           |                   |
           +-------------------+
```

虚线以下都是从普通进程向文件系统进程执行读取请求的机制.开始,`read`(JOS提供的)可以在任何文件描述符上工作,并简单地分派到适当的设备读取功能,通常是`devfile_read`(当然可以有更多的设备类型,比如pipe).`devfile_read`实现专门针对磁盘上文件的读取.`lib/file.c`中的`devfile_*`函数实现了文件系统操作的客户端,并且都以大致相同的方式工作,将参数捆绑在请求结构中,调用`fsipc()`发送进程间通信请求,并解包和返回结果.`fsipc()`函数只处理向服务端发送请求和接收回复的常见细节.

文件系统服务端代码在`fs/serv.c`中.它在`serve()`函数中循环,通过IPC不断接收请求,将该请求发送到适当的处理函数,并通过IPC发回结果.在`read`示例中,`serve`将分派给`serve_read`,它将处理特定于读取请求的IPC细节,例如解开请求结构,最后调用`file_read`来实际执行文件读取.

回想一下,JOS的IPC机制允许一个进程发送一个32位数,并且可以选择共享一个内存页面.为了从客户端向服务器发送请求,我们使用32位数来表示请求类型(文件系统服务端提供的RPC均被编号,就像syscalls被编号一样),并将请求的参数存储在通过ipc共享的页面上的`union Fsipc`中.在客户端,我们总是共享`fsipcbuf`所在页面;在服务器端,我们将传入的请求页面映射到`fsreq(0x0ffff000)`.

服务器也会通过IPC发回响应.我们使用32位数作为函数的返回代码.对大多数RPCs来说,通常只需要返回32位数即可.`FSREQ_READ`和`FSREQ_STAT`需要额外返回数据,它们只需将数据写入客户端发送请求的页面,没有必要在响应IPC中再次发送此页面,因为客户端已经与文件系统服务端共享此页面.此外在`FSREQ_OPEN`中,服务端将与客户端共享一个新的"Fd页面",从而返回文件描述符.

### Excrcise5
在`fs/serv.c`中实现`serve_read`.

`serve_read`的繁重工作将由`fs/fs.c`中已经实现的`file_read`来完成(当然这也只是对`file_get_block`的一系列调用).`serve_read`只需要为文件读取提供RPC接口.查看`serve_set_size`中的注释和代码,大致了解应该如何完成服务端功能。

使用`make grade`测试我们的代码.代码应该通过`serve_open/file_stat/file_close`和`file_read`,且得分为70/150.

### Exercise6
在`fs/serv.c`中实现`serve_write`,在`lib/file.c`中实现`devfile_write`.

使用`make grade`测试我们的代码.代码应该通过`file_write`,`file_read after file_write`,`open`和`large file`,且得分为90/150.

---
# Spawning Processes 创建子进程
JOS已经为我们提供了用于`spawn()`函数(参见`lib/spawn.c`),它创建一个子进程,将代码镜像从文件系统加载到子进程中,然后启动运行该程序的子进程.父进程然后独立于子进程继续运行.`spawn()`函数实际上类似于UNIX中`fork()`,然后在子进程中立即执行`exec()`.

我们实现了`spawn()`而不是UNIX风格的`exec()`,因为`spawn()`更容易从用户空间以"微内核"的方式实现,而无需内核的特殊帮助.想想看为了在用户空间实现`exec()`,你必须做些什么,并确保你理解为什么会更难.

### Exercise7
`spawn()`依赖于新的系统调用`sys_env_set_trapframe()`来初始化新创建进程的状态.在`kern/syscall.c`中实现`sys_env_set_trapframe`.(不要忘记在syscall()中调度新添加的系统调用).

通过从`kern/init.c`运行`user/spawnhello`程序来测试我们的代码,该程序将尝试创建子进程,并运行文件系统中的`/hello`.

使用`make grade`测试您的代码.

## Sharing library state across fork and spawn 通过fork和spawn共享函数库
UNIX文件描述符是一个通用概念,不仅包括普通文件,目录文件,也包括管道,console I/O等.在JOS中,这些设备类型都有一个对应的`struct Dev`,带有针对该类型实现`read/wite`等函数的指针.`lib/fd.c`在此基础上实现了通用的UNIX-like文件描述符接口.每个`struct Fd`指示其设备类型,`lib/fd.c`中的大多数函数只是将操作分派给对应的`struct Dev`中的函数.

`lib/fd.c`还在每个用户进程的地址空间中维护了一个文件描述符表,起始地址为`FDTABLE`.应用程序当前最多可以打开32个文件描述符,文件描述符表为此保留了一页的内存地址空间(4KB).当且仅当相应的文件描述符正在使用时,特定的文件描述符表页才会被映射.每个文件描述符从`FILEDATA`开始也有一个可选的"data page",如果需要,可以使用这个数据页.

我们希望通过`fork`和`spawn`可以共享文件描述符,但是文件描述符是保存在用户空间的.现在`fork`时,内存将被标记为`copy-on-write`,因此文件描述符将被复制而不是共享.(这意味着进程将无法在自己没有打开的文件中查找,pipe也无法在fork中使用.)而`spawn`时,内存根本不会被复制.(实际上,子进程根本就没有打开的文件描述符.)

我们将修改`fork`,因为某些内存区域保存着"系统库",并且应该在所有进程间共享.我们将在`page table entry`中使用一个未使用的位,而不是在某些内存区域硬编码(就像我们在fork中使用PTE_COW).

我们在`inc/lib.h`中定义了一个新的`PTE_SHARE`位.该位是英特尔和AMD手册中标记为"保留给软件使用"的三个PTE位之一.我们将设置一个规定,如果一个`page table entry`设置了这个位,那么PTE应该在`fork`和`spawn`中从父进程直接复制到子进程.请注意,这不同于将其标记为`copy-on-write`:如前所述,我们希望确保所有进程始终共享同一页面.

### Exercise8
在`lib/fork.c`中修改`duppage`以符合新的规定.如果`page table entry`设置了PTE_SHARE位,直接将其复制到子进程中.(我们应该使用`PTE_SYSCALL`而不是`0xfff`来屏蔽`page table entry`中的相关位.因为`0xfff`也会获取`accessed`和`ditry`位.)

同样,在`lib/spawn.c`中实现`copy_shared_pages`.它应该遍历当前进程中的所有`page table entry`(就像fork一样),并将所有设置了`PTE_SHARE`位的`pte`复制到子进程中.

使用`make run-testpteshare`测试我们的代码是否运行正常.应该会看到"fork handles PTE_SHARE right"和"spawn handles PTE_SHARE right".

使用`make run-testfdsharing`检查文件描述符是否正确共享.应该会看到"read in child succeeded"和"read in parent succeeded".

---
# The keyboard interface
为了让shell正常工作，我们需要一种方式来向它输入信息.QEMU一直在显示CGA显示器和串行端口的输出,但是到目前为止,我们只在`kernel monitor`中进行输入.在QEMU中,图形窗口中键入的输入作为从键盘到JOS的输入,而console中键入的输入将作为串行端口上的字符.`kern/console.c`已经包含了自lab1以来`kernel monitor`一直使用的键盘和串行驱动程序,但是现在我们需要将它们应用到系统的其余部分.

### Exercise9
在`kern/trap.c`中,调用`kbd_intr`处理中断`IRQ_OFFSET+IRQ_KBD`,调用`serial_intr`处理中断`IRQ_OFFSET+IRQ_SERIAL`.

JOS在`lib/console.c`中已经实现了console设备的读写函数.`kbd_intr`和`serial_intr`用最近读取的输入填充缓冲区,而控制台文件类型将读取缓冲区(默认情况下,控制台文件类型用于stdin/stdout,除非用户重定向它们).

通过运行`make run-testkbd`并键入几行来测试我们的代码.当我们完成时,系统应该把我们的输入打印出来.请尝试同时在控制台和图形窗口输入一些信息.

---

# The Shell
运行`make run-icode`或`make run-inode-nox`.这将启动JOS内核并运行`user-icode`.`icode`执行`init`,它将console设置为文件描述符0和1(标准输入和标准输出).`init`将会创建子进程`sh`,即shell.我们应该能够运行以下命令:
```
	echo hello world | cat
	cat lorem |cat
	cat lorem |num
	cat lorem |num |num |num |num |num
	lsfd
```

请注意,用户库函数`cprintf`将直接打印到console,而不使用文件描述符.这对于调试来说很好,但是对于pipe连接到其他程序来说就不太好了.要将输出打印到特定的文件描述符(例如,标准输出1),请使用`fprintf(1,"...",...)`.`printf("...",...)`是打印到`FD 1`的缩写.有关示例,请参见`user/lsfd.c`.

### Exercise10
当前shell不支持IO重定向.最好可以运行`sh < script`,而不是像上面那样手工输入脚本中的所有命令.在`user/sh.c`添加重定向命令`<`.

通过输入`sh <script`来测试我们的代码.

运行`make run-testshell`来测试shell.`testshell`只是将上述命令(也可以在`fs/testshell.sh`中找到)输入到shell中,然后检查输出是否与`fs/testshell.key`相匹配.

---

# End
lab5到这里全部结束,可以通过`make grade`命令来测试代码.如果没有错误的话,应该可以全部通过.














---
