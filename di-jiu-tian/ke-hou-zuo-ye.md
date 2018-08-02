# xv6 锁实现
## 实验
```
  struct spinlock lk;
  initlock(&lk, "test lock");
  acquire(&lk);
  acquire(&lk);
```
Q. 执行上面代码时,将会发生什么现象?
A. 两次调用`acquire()`时,在第二次调用将会检查,导致走到`panic("acquire")`.线程将会退出.

## 实验: ide.c中的中断
1. 在xv6中,我们使用了`idelock`来保护`idequeue`.
2. `idelock`是一个自旋锁,而我们知道在自旋锁中,中断是被关闭的.
3. 如果在`ide.c/iderw()`中,`acquire()`之后加入`sti()`,`release()`之前加入`sti()`.重新编译运行xv6,查看现象.
4. 现象: xv6启动时发生了panic,查看错误信息发现是因为重复获取锁.
5. 查看`acquire()`代码中的`holding()`函数的判断逻辑,发现自旋锁是以CPU持有的.
6. 换句话说,Core A上的线程T1持有自旋锁L后,该自旋锁L将会和Core A相绑定.
7. 如果此时线程T1被调度,Core A运行一个新的线程T2,而T2同样去获取锁L的话,将会出现`panic acquire`.因为在此之前Core A已经持有了锁L,并且还未释放.
8. 即使没有新的线程在Core A上去尝试获取锁L,如果线程T1被调度到另外一个Core B上去运行,在T1尝试释放锁L时,将会发生`panic release`.因为此时锁L所绑定的Core是Core A.

## file.c中的中断
1. 前一个实验中,我们修改了`idelock`的获取和释放,结果发现系统基本无法启动.
2. 这个实验中,我们对`file.c`中的`file_table_lock`,重复这样的实验,观看一下现象.
3. `file_table_lock`这个锁主要是用于保护`ftable`这样一个数据结构.其中保存了整个系统可以打开的文件总数.
4. 修改函数`filealloc()`在`acquire()`之后增加`sti()`.在两处`release()`之前增加`cli()`.还需要增加头文件`x86.h`.
5. 重新编译运行xv6.
6. 一般来讲,xv6将不会崩溃,请解释为什么?查看`filealloc()`的调用者,可能会有帮助.
7. 查看源码发现,`filealloc()`主要是由`open`和`pipe`系统调用来使用.因此在xv6启动过程中不会发生什么意外.
8. 可以预见,如果发生多个线程同时打开文件,那么就有条件竞争的可能.后果可能导致多个线程使用了同一个`file`结构体.导致打开文件出现意想不到的问题.

## xv6 锁实现
Q: 在xv6的锁视线中,`release()`为什么要在清理`lk->locked`之前,清理`lk->pcs[0]`和`lk->cpu`?为什么可不可以挪到后面?
A: 显然是不行的.因为`lk->locked`一旦释放,那么锁就可能被其他线程抢走,在其他线程抢走锁之后,再修改锁的内容无疑是非常危险的.