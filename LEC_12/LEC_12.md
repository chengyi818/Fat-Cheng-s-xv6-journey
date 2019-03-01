# LEC 12 文件系统

## 课程大纲
1. 文件系统
2. 磁盘和磁盘布局
3. xv6案例分析

## 文件系统的作用
1. 在重启计算机过程中,持久化存储.
2. 命名和管理文件
3. 在不同进程和用户间共享文件.

## 文件系统迷人的特点
1. 崩溃恢复
2. 性能
3. 文件共享
4. 安全
5. 概念抽象: 管道, 设备, /proc等

## 文件系统相关API
1. UNIX/Posix/Linux/xv6/&c
2.
```
  fd = open("x/y", -);
  write(fd, "abc", 3);
  link("x/y", "x/z");
  unlink("x/y");
```

## 文件系统中的一些高级API
1. 通过文件file这个抽象来描述对象
2. 通过人类可读的文件名来命名文件,而不是通过对象的id.
3. 通过命名的层次结构来组织文件.
4. 通过分层,尤其是log层,隐藏了文件同步的细节.

## 关于API的一些解释
1. fd总是指向一个`struct file`结构体,即使file对应的文件改名或者被删除,这一点也不会改变.
2. 可能同时存在很多链接指向磁盘上同一个文件,比如通过link,不同目录上的路径指向同一个文件.因此磁盘上的文件需要独立的计数器来记录引用个数.
3. 文件系统通过磁盘上的`struct dinode`结构体来保存`inode`信息.
4. 文件系统通过`inode-number`来定位`inode`的位置.
5. `inode`必须有路径指向计数器,以便我们可以在合适的时机释放`inode`.
6. `inode`必须有c指针指向的计数器,以便我们可以在内存中释放该`inode`缓存.
7. 当一个文件既没有路径指向,且没有c指针指向时,我们就会从磁盘将该文件删除.

---

# xv6

## 文件系统软件分层

```
+-------------------+
|  system calls     |
+-------------------+
|  name ops, FD ops |
+-------------------+
|  Inode            |
+-------------------+
|  Inode cache      |
+-------------------+
|  Logging          |
+-------------------+
|  Buffer cache     |
+-------------------+
|  Disk driver      |
+-------------------+
```

## IDE
1. IDE是CPU和硬件通信的一种[标准](https://en.wikipedia.org/wiki/Parallel_ATA)
2. CPU和IDE控制器通信,而IDE控制器和硬件通信,比如硬盘,SSD,CD-ROM等.

## 硬盘驱动hard disk drives(HDD)
1. 这部分关于硬盘的物理结构介绍,可以参考[鸟哥的Linux私房菜](http://cn.linux.vbird.org/linux_basic/0130designlinux_2.php)


## 固态硬盘驱动solid state drives(SSD)
1. 非易失性闪存
2. 随机访问: 100 毫秒
3. 顺序读写: 500 MB/s
4. flash重新写入前,必须擦除.
5. 闪存的写入次数有限.

## HDD和SSD的共同特点
1. 顺序访问远快于随机访问
2. 大文件的访问快于小文件
3. 这个物理特性对文件系统的设计和性能有着非常大的影响.

## disk blocks
1. 对大多数OS而言,block的大小总是扇区sector的倍数.通常block 4KB = 8 sector.
2. 这种大小减少了记录和查找的消耗.
3. 为了简化实现,xv6中一个block就是一个扇区的大小,即512 byte.

## 磁盘布局
1. xv6文件系统位于第二个IDE驱动上,第一个IDE驱动上仅保存着内核.
2. xv6将IDE驱动抽象成一个扇区数组,忽略了`track structure`
3.
```
  0: unused
  1: super block (size, ninodes)
  2: log for transactions
  32: array of inodes, packed into blocks
  58: block in-use bitmap (0=free, 1=used)
  59: file/dir content blocks
  end of disk
```

## 磁盘初始化
1. xv6的`mkfs`根据上面的布局,创建了一个新的空文件系统.
2. 整个磁盘布局在初始化完成后,不会再变化.

## 元数据meta-data
1. 磁盘上除了文件内容的信息都可以被称为元数据
2. 元数据包括: super block, i-node数组, bitmap, directory信息.

## 磁盘上的inode
1. 通过阅读xv6 book,我们可以知道inode可以分为两种,一种存在于磁盘上,一种存在于内存中.
2. type: (free, file, directory, device)
3. nlink: 表示有多少路径指向该inode
4. size: inode的大小
5. addrs[12+1]: 表示文件内容所在的inode位置,包括12个直接block和1个间接blocks地址数组.
6. 最大支持文件大小为(12 + (512/4))*512 =6KB+64KB = 70KB

## inode举例说明
1. 如何找到一个文件的第8000个字节?
2. 逻辑block number为: 8000/512 = 15.625
3. 因此第8000个字节位于间接blocks数组的第4个entry.

## inode id
1. 每个inode都有一个inode number
2. inode number即inode在磁盘inode数组的索引.
3. 每个inode的大小为64字节.
4. 通过inode number很容易找到inode所在的位置.
5. 每个indoe在磁盘的地址: 32*512 + 64*inum.

## 文件系统目录的实现
1. 目录和文件的实现相似,当用户不能直接读写目录.
2. 目录对应的inode中的内容是`struct dirent`数组
3. `struct dirent`由inum和14字节的文件名组成.
4. 当inum为0时,dirent是空闲的.


## 文件系统抽象
1. 我们应当将文件系统看作一个磁盘上数据结构组成的一棵树.
2. 文件数由三个部分组成: dirs, inodes, blocks.
3. 此外,还有两个内存中的缓存池: inodes, blocks.

---

## xv6文件系统的实际应用
1. 我们重点关注磁盘写入.
2. 展示了磁盘上的数据结构如何更新

## xv6创建文件的过程
1. 执行`echo > a`.
2. 调用路径:
```
call graph:
  sys_open      sysfile.c
    create      sysfile.c
      ialloc    fs.c                    // 写入block 34
      iupdate   fs.c                    // 写入block 34
      dirlink   fs.c
        writei  fs.c                    // 写入block 59
```

### Q&&A
1. block 34中保存的内容是什么?
> block 34是inode数组,我们将从中找到空闲的inode结构体.

2. 为什么要两次写入block 34?
> 第一次写入是从中分配了一个空闲inode结构体,第二次写入是将更新后的inode信息写入磁盘

3. block 59的内容是什么?
> 根目录的inode节点所对应的内容block.我们创建了文件,需要更新所在的目录信息.

4. 并行访问ialloc()是否会引起冲突?
> 不会,bread()->bget()返回的buf已经加锁.

## xv6写入文件的过程
1. 执行`echo x > a`
2. 调用路径:
```
call graph:
  sys_write       sysfile.c
    filewrite     file.c
      writei      fs.c
        bmap
          balloc                      // 写入block 58
            bzero                     // 写入block 508
        iupdate                       // 写入block 34
```

### Q&&A
1. block 58中保存的内容是什么?
> 标志着所有block使用情况的bitmap

2. block 508中保存的内容是什么?
> 文件a中内容所保存的block

3. 为什么需要调用iupdate()?
> 文件的长度和保存内容的block等信息需要更新到文件对应的inode中

4. 为什么会两次调用`writei() + iupdate()`?
> echo会两次调动write(),第一次写入内容,第二次写入换行符


## xv6删除文件
1. 执行`rm a`
2. 调用路径:
```
call graph:
  sys_unlink
    writei                              // 写入block 59, 目录内容
    iupdate                             // 写入block 34, 文件的链接数
    iunlockput
      iput
        itrunc
          bfree                         // 写入block 58
          iupdate                       // 写入block 34
        iupdate                         // 写入block 34
```

### Q&&A
#### block 59中的内容?
> 目录中存在文件a的链接,需要删除

#### block 34中的内容?
> 文件a对应的inode节点, 需要移除一个路径指向计数器

#### block 58中的内容?
> blocks bitmap, 移除文件a所占用的block

#### 为什么需要3次iupdate()?
1. 查看bio.c中关于block cache的代码.
2. block cache缓存了一些最近使用的blocks
3. `bcache`的定义在bio.c文件的头部
4. 文件系统FS调用`bread()`,`bread()`调用`bget()`
5. `bget()`首先查看目标block是否被缓存
6. 若已被缓存,且block锁未被占用,则获取锁后,返回目标block.
7. 若已被缓存,且block锁正被占用,则睡眠等待锁释放.
8. 若未被缓存,则分配一个新的buffer缓存用于载入block的内容.
9. `b->refcnt++`保证了在我们等待buffer的过程中,该buffer不会被其他block使用.
10. 这里有两层锁的使用.
11. `bcache.lock`保护了整个block cache.
12. `buf->lock`保护了单个buf.

#### block cache的替换策略是什么?
1. 整个block cache结构: `prev...head...next`
1. `bget()`首先尝试从head向后查找block是否被缓存?
2. 若未被缓存,`bget()`尝试从链表尾部找一个未被使用的buffer
3. `brelse()`释放buffer,将buffer移动到链表头部.即最近使用的在链表头部.

#### 这个替换策略是最优的么?

#### 若同时有多个进程尝试读取磁盘?顺序是怎么样的?
1. `iderw()`会将ide读写请求放到一个请求队列中
2. `ideintr()`每次都会执行队头的请求
3. 因此,顺序是先进先出FIFO

#### FIFO是一个好的磁盘调度顺序么?
1.















































----
