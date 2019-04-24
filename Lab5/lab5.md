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

### File Meta-data
描述文件系统中文件的元数据的布局由`inc/fs.h`中的`struct File`描述.该元数据包括文件的名称,大小,类型(常规文件或目录),以及指向组成文件的块的指针.如上所述,我们没有索引节点,因此元数据存储在磁盘的目录条目中.与大多数“真实的”文件系统不同,为了简单起见,我们将使用`struct File`来表示文件元数据,因为它同时出现在磁盘和内存中.

`struct File`中的`f_direct`数组包含存储文件的前10个块的块号的空间,我们称之为文件的直接块.对于大小高达10*4096 = 40KB的小文件,这意味着文件所有块的块号将可以直接保存在`struct File`本身.然而,对于较大的文件,我们需要一个存放文件其余块号的地方.因此,对于任何大于40KB的文件,我们会分配一个额外的磁盘块,称为文件的间接块,以容纳最多4096/4 = 1024个额外的块号.因此,我们的文件系统允许文件的大小达到1034个块,或者仅仅超过4兆字节.为了支持更大的文件."真实"文件系统通常也支持双间接和三间接块.

![file](../images/file.png)

### Directories versus Regular Files

---

# The File System
## Disk Access
## The Block Cache
## The Block Bitmap
## File Operations
## The file system interface

---

# Spawning Processes
## Sharing library state across fork and spawn

---

# The keyboard interface

---

# The Shell














---
