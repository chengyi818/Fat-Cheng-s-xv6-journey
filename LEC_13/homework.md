# homework

## 大纲
本作业我们将会修改xv6的log,内容分为两个部分:
1. 首先,我们会人为造成系统crash,以此证明log机制的必要性.
2. 其次,我们将会提升xv6 logging系统的性能

## 人为crash
xv6日志的目的是使所有的文件系统操作对于崩溃都是原子的.比如创建文件,我们不仅需要在directory inode中添加一个新的entry,还需要创建一个新的inode节点.
如果没有日志系统,crash发生在第一步操作完成,第二步还未完成时,将导致文件系统处于错误状态.


### 修改xv6代码
下面我们来模拟一个文件系统crash的情况:

首先,使用如下代码替换`log.c`中的`commit()`.
```
#include "mmu.h"
#include "proc.h"
void
commit(void)
{
  int pid = myproc()->pid;
  if (log.lh.n > 0) {
    write_log();
    write_head();
    if(pid > 1)            // AAA
      log.lh.block[0] = 0; // BBB
    install_trans();
    if(pid > 1)            // AAA
      panic("commit mimicking crash"); // CCC
    log.lh.n = 0;
    write_head();
  }
}
```
BBB行导致本来应该写的第一个block变为写block 0.`log.lh.block[0]`应该修改disk inode数组,将一个inode标记为used.BBB行导致本应写到inode数组的内容写到了block 0.这样inode仍然显示free.CCC行强制panic.另外AAA行确保了init进程可以运行正常.

其次,用以下代码替换`log.c`中的`recover_from_log()`:
```
static void
recover_from_log(void)
{
  read_head();
  cprintf("recovery: n=%d but ignoring\n", log.lh.n);
  // install_trans();
  log.lh.n = 0;
  // write_head();
}
```

最后,我们将`Makefile`中,`QEMUEXTRA = -snapshot`注释掉.

### 测试
至此,我们的修改完成.下面我们来模拟一次文件系统crash,操作步骤如下:
1. $make clean;make qemu
2. echo hi > a

现在你应该看到xv6已经crash了,这模拟了在一个没有日志功能的系统上,在创建文件中途crash的情况.

现在我们再次运行`make qemu`,然后执行`ls`或者`cat a`.
我们会发现系统再次crash,并输出`panic: ilock: no type`.

### 思考
Q: 思考一下为什么会出现这个打印?另外回想一下创建一个文件的过程,有哪些内容被写到了磁盘,哪些没有写入?

A: 与正确的创建文件相比,我们唯一没有写入的就是文件a对应的inode的状态.当我们再次重启后,根目录中已经记录了a对应的inode-number,但是a对应的inode type并没有修改.当我们尝试读取inode a的内容时,`ilock`会检查dinode a的type,发现为`0`,最终会抛出panic.

## 解决问题
我们恢复`recover_from_log()`:
```
static void
recover_from_log(void)
{
  read_head();
  cprintf("recovery: n=%d\n", log.lh.n);
  install_trans();
  log.lh.n = 0;
  write_head();
}
```

不要删除fs.img,我们再次运行`make qemu`,并执行`cat a`.
我们会发现不会crash了,并且会有如下打印:`recovery: n=2`.

最后,恢复`commit()`的代码,并再次`make clean`.恢复xv6的代码

### 思考
1. 为什么现在不会crash了?
因为丢失的修改信息已经保存到了log,`recover_from_log()`将修改重新写入了正确位置.

2. 为什么文件a的内容为空?
因为`echo hi > a`,包含了多步文件操作,在创建文件a时已经crash.后面写入内容的指令当然没有执行.


## 精简commit
假设文件系统想要更新block 33的内容.
1. 文件系统需要调用`bp=bread(block 33)`,然后更新bp的内容.
2. `commit()/write_log()`将会将block 33的内容写入磁盘log区,假设是block 3.
3. `commit()/install_trans()`首先将block 3的内容从磁盘中读出.然后将内容拷贝到block 33的内存镜像.最终将block 33的内存镜像写回磁盘.

然而在`install_trans()`中,block 33在写回磁盘前,都被标记为DIRTY,因此不会被buffer cache回收.

因此在`install_trans()`中,我们无需从log区中,读取`block 33`的内容.

### 修改
我们需要修改`log.c`,当`install_trans()`是从`commit()`中调用过来时,我们不需要从log区中读取.

### 测试
在xv6中创建一个文件,重启xv6,确认文件依然存在.
