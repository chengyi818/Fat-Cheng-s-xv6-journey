# 进程, 线程和调度

## 大纲
1. 课后作业
2. 线程切换
3. 调度

---

## 课后作业
homework 9深入思考

### iderw()
1. 代码目录`xv6/ide.c`

2. `idelock`保护了什么?
    整个系统中可能存在很多进程,每个进程都可以发起ide请求,而ide驱动每次只能处理一个请求.因此需要一个`idequeue`用于缓存这些请求.如果没有`idelock`,那么缓存ide请求时就会发生竞争,导致部分ide请求永远得不到处理.同时发起请求的线程将永远处理sleep状态.

3. 在acquire之后增加sti(),在release之前增加cli(),将会发生什么?为什么?
答:  现象为可能出现`panic acquire`
    原因: 改动将导致在处理ide请求是,允许发生中断,即当前进程可以被调度.如果线程A获取idelock在core 1上,在线程A sleep之前是可能被调度出去的.如果另一个线程B此时调度到core1上运行,且同时发起了ide请求,那么就会尝试获取idelock.此时idelock处于被锁状态且idelock->cpu恰好为core 1,因此会直接panic.

4. 如果自旋锁抢锁时,没有检查锁是否已被当前core获取,会发生什么?
答:  如果没有已获取检查,设想这样一种情况,线程A运行在core 1上,且此时线程A已获取自旋锁a.若此时线程A再次尝试获取自旋锁a,那么
`while(xchg(&lk->locked, 1) != 0)`将死循环.且线程A既不会被调度也不会释放锁a.

5. 在原代码中,中断是如何发生和处理的?
答: ide操作和CPU运行速度相比要慢的多,因此这部分处理主要使用了异步处理的方式.
    对进程而言通过调用`iderw()`来将ide请求添加到`idequeue`中,然后线程将进入睡眠等待状态.
    对ide驱动依次从`idequeue`中取出ide请求,并发送给硬件,然后CPU将返回,并运行其他进程.
    当硬件完成ide请求后,将发出硬件中断`T_IRQ0 + IRQ_IDE`,并触发中断处理.中断处理将会调用`ideintr()`来读写数据,并唤醒之前睡眠的进程.若`idequeue`中还有尚未完成的请求,将向硬件发出新的请求.
    
6. 对当前代码而言,IDE中断会被发送到同一个core上.若IDE中断发生在不同的core上会如何?
提示:
    1) 阅读`spinlock.c acquire()/release()`,它们是如何控制中断的?
        `acquire()`时将会在当前core上禁用中断,然后不断轮询直至获取锁.在此期间不会被调度.
        `release()`释放锁之后,将会启用中断.
    2) 为什么要使用`pushcli()/popcli()`对中断控制进行计数?
        通过对中断控制的计数可以确保禁用和启用中断是成对出现的.不会出现预料之外的情况.比如两次`cli()`,一次`sti()`,此时中断已启用,未必符合预期.
   3)  `mycpu()->intena`表征了`pushcli`前,core是否允许中断.若调用`pushcli()`前禁用中断,那么`popcli()`也不会启用中断,不会出现
   
答: 看不出什么问题.好像在`idelock`的保护下没啥问题?

7. 为什么`acquire()`要在抢占锁之前,禁用中断?

答: 从注释来看,主要是为了避免死锁.如果未禁用中断,则表示在进入`acquire()`后,仍然可以被调度.设想这样一个场景: 线程A运行在core a上且获取了锁,线程B运行在core b上正在等待锁.此时如果线程B被调度到core a上,线程A被调度到core b上,在线程A释放锁时,线程A会panic.此时线程B将永远无法获取锁.

### filealloc()
1. 代码目录`xv6/file.c`

2. 在`filealloc()`中,如果中断启用了会怎样?
答: 中断启用表示可以进程调度,那么在`release(&ftable.lock)`会发生panic.

3. 锁`ftablc.lock`保护的什么资源?
答: 在xv6中,ftable是一个用于管理file资源的结构体,所有file资源在初始化被申请,且可以循环使用.`ftable.lock`主要就是用于保护这些file资源.

4. 即使我们修改了`filealloc()`代码,为什么通常不会出问题?
答: 之前我们已经提到过了,想要发生问题的前提是在获取锁之后,发生进程调度.但是这个窗口很小,因此大部分时候不会触发.

5. 如何使问题必现?
答: 根据前面的分析,只要在启用中断,发生进程调度即可.因此可以在`acquire()`之后,添加`yield()`,强制进程调度.

---

## 进程调度 

### 进程
一种抽象的虚拟机,好像拥有独立的CPU和内存,不会被其他进程所影响,进程间互相隔离.

### 进程相关API
```
fork
exec
exit
wait
kill
sbrk
getpid
```

### 挑战: 进程数大于CPU核数
1. 设想这样一个场景: CPU仅有2个核,而同时需要运行的进程有3个.

2. 狼多肉少,因此需要复用这些CPU.相关概念包括: 时分,调度,上下文切换.

### xv6的解决方法
1. 每个进程包括一个用户态线程和一个内核态线程
2. 每个CPU拥有一个调度器线程
3. 支持多CPU

### 什么是线程?
1. 一个CPU执行状态的集合,包括寄存器和栈.

### xv6进程切换概览
```
  user -> kernel thread (via system call or timer)
  kernel thread yields, due to pre-emption or waiting for I/O
  kernel thread -> scheduler thread
  scheduler thread finds a RUNNABLE kernel thread
  scheduler thread -> kernel thread
  kernel thread -> user
```   
1. 用户态线程通过系统调用或者时钟中断进入内核态线程
2. 内核态线程因为抢占或者IO等待,让出CPU
3. 内核态线程切换到调度器线程
4. 调度器线程确定一个处于RUNNABLE状态的内核态线程
5. 调度器线程切换到新的内核态线程
6. 内核态线程切换到新的用户态线程

### xv6进程状态定义
1. RUNNING 正在执行
2. RUNNABLE 等待执行
3. SLEEPING 睡眠等待
4. ZOMBIE 僵尸状态,等待回收
5. UNUSED 处于进程池中,等待被使用

### 注意:
1. xv6中很多内核态线程共用相同的内核地址空间,即使用相同的内核态页表.
2. xv6中,每个进程仅有一个用户态线程
3. 很多现代OS,比如Linux,支持多线程.即一个进程对应多个用户态线程.

### 上下文切换是xv6中最困难的事之一
1. 多核处理
2. 锁
3. 中断处理
4. 进程终止

---

##  进程切换分析
场景设置如下: 两个进程同时运行在单核CPU,进程将会发生切换.

### 时钟中断
1. 这里主要需要了解的是,何时会发生进程切换?
2. 主要代码位于`trap.c`,如下.当发生时钟中断时,会强制当前进程让出CPU,进行一次进程调度.
```
  // Force process to give up CPU on clock tick.
  // If interrupts were on while locks held, would need to check nlock.
  if(myproc() && myproc()->state == RUNNING &&
     tf->trapno == T_IRQ0+IRQ_TIMER)
    yield();
```
3. `yield()`会将当前进程状态改为`RUNNABLE`,当前进程会等待下次被调度运行.
4. 然后将执行`sched()`,首先将进行一系列检查,核心代码如下.
```
  swtch(&p->context, mycpu()->scheduler);
```

### swtch: 切换到调度器上下文
1. 什么是上下文context? 上下文本质上一组内核线程的寄存器状态.
2. xv6的context一直存在于栈上
3. 上下文指针总是指向%esp栈顶
4. 用户线程的寄存器状态是以`trapframe`的形式保存于栈中.
5. `proc.h`中,context的定义:
```
// Saved registers for kernel context switches.
// Don't need to save all the segment registers (%cs, etc),
// because they are constant across kernel contexts.
// Don't need to save %eax, %ecx, %edx, because the
// x86 convention is that the caller has saved them.
// Contexts are stored at the bottom of the stack they
// describe; the stack pointer is the address of the context.
// The layout of the context matches the layout of the stack in swtch.S
// at the "Switch stacks" comment. Switch doesn't save eip explicitly,
// but it is on the stack and allocproc() manipulates it.
struct context {
  uint edi;
  uint esi;
  uint ebx;
  uint ebp;
  uint eip;
};
```

6. `swtch(from, to)`. 将当前线程的寄存器压栈,并将%esp保存到`from`中.
然后从`to`中获取%esp,并将寄存器出栈,最后调用ret获取新的eip.从而完成线程切换.

### scheduler() 调度器线程
1. 在上一次`scheduler()`调度了一个进程时,执行代码为`swtch(&(c->scheduler), p->context);`.此时`eip`指向下一条指令地址.
2. 当切换到`scheduler()`后,第一条执行的代码为`switchkvm();`
3. 首先,将会切换为内核页表,将当前CPU执行的进程置空.
4. 其次,将会进入for循环,寻找到一个`RUNNABLE`进程.然后调用`switchuvm()`设置TSS和页表.
5. 最后,再次调用`swtch()`,从调度器线程切换到新进程的内核线程.
6. 将切换完成后,`scheduler()`线程的栈顶又是一个context.等待下一次执行调度任务.
7. 对于新进程而言,将从`swtch(&p->context, mycpu()->scheduler);`后开始运行.

#### Q&&A
1. xv6的调度策略是怎样的?
答: 当前调度策略很简单,就是从进程管理结构体`ptable`的头部向尾部遍历,执行第一个`RUNNABLE`进程.

2. 刚刚执行完的进程,有可能被立即调度么?换言之,是否存在进程饿死的现象?
答: 有可能被立即调度执行,即存在进程饿死的现象.比如old进程占据了`ptable.proc[]`










































---