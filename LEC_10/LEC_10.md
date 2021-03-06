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

### Q&&A
#### xv6的调度策略是怎样的?
答: 当前调度策略很简单,就是从进程管理结构体`ptable`的头部向尾部遍历,执行第一个`RUNNABLE`进程.

#### 刚刚执行完的进程,有可能被立即调度么?换言之,是否存在进程饿死的现象?
答: 有可能被立即调度执行,即存在进程饿死的现象.比如old进程占据了`ptable.proc[0]`的位置.那么在old进程执行完之前,将不会有其他进程可以被调度到.

#### 在`scheduler()`中,为什么释放了`ptable.lock`后,很快再次获取?这样做的目的是什么?
答: 假设这样一个场景: 进程A正在core a上运行,core b上没有进程在运行,因此在`scheduler()`中不断循环.此时进程A执行了fork()指令,生成了进程C.如果core b上没有这样一个快速释放获取的过程,那么进程A将无法获取一个可用的proc资源.
换言之,给其他进程一个使用`proc table`的机会,不然在运行进程比核数少的情况下,将会发生死锁.

#### 为什么在`scheduler()`中,会短暂启用中断?
答: 假设这样一个场景: 当前ptable中没有可以运行的进程,因此core处于循环中.此时存在进程由于等待IO操作,正处于SLEEPING.因此我们需要给device设备一个时间窗口,来唤醒这些等待的进程.
 
#### 为什么在`yield()`中获取的`ptable.lock`在`scheduler()`中释放?
答: 通常而言,获取锁和释放锁是在同一个线程之中.但是在线程切换中,获取锁和释放锁是在不同的线程中.比如获取锁是在old线程的`yield()`,而一种释放的可能是在`scheduler()`线程中.

假设这样一个场景: 我们在同一个线程中申请和释放锁.进程A在core a上执行`yield()`.进程A的状态被修改为`RUNNABLE`.此时core b正在执行`scheduler()`.而此时进程A的context甚至还没有正确生成.

换言之,`swtch()`操作必须是一个原子操作,必须处于锁的保护之下.

`sched()`和`scheduler()`虽然是两个线程,但却存在一种**协程**关系:
* 调用者知道`swtch()`后,将会切换到哪里.
* 被调用者知道`swtch()`从哪来
* 两者通过`ptable.lock`来协同工作
* 一般的线程切换并没有这样的先后顺序

#### 我们如何确定`scheduler()`线程已经准备好让我们`swtch()`进入?除了`swtch()`,还有其他地方会进入`scheduler()`线程么?
答: kernel初始化完成后,会在`main()`中调用`mpmain()-->scheduler()`,是首次调用.

`ptable.lock`保护了线程上下文的完整性,有如下场景:
1. 如果进程处于RUNNING状态,那么上下文保存在CPU的寄存器中.
2. 如果进程处于RUNNABLE状态,那么位于栈顶的上下文context保存了寄存器的值.
3. 如果进程处于RUNNABEL状态,CPU不会使用进程相关内核线程栈,因此不会触碰保存好的context信息.

在使用`yield()`进行进程调度的过程中,我们使用了如下方式确保了进程切换的原子性:
1. 全程处于`ptable.lock`的保护之下
2. 禁用中断,所以时钟中断不会打断`swtch`的过程.
3. 其他CPU无法在`swtch()`的过程中,执行`scheduler()`

#### 内核线程是否支持抢占?在内核线程执行过程中,发生时钟中断会如何?内核线程栈的结构?
答: 当前我们的调度策略是通过轮询的形式,并没有引入优先级的概念,因此是不支持抢占的.

内核线程执行过程中,发生时钟中断分两种情况,若没有禁用中断,则进入时钟中断处理,触发`yield()`,进行进程调度.若禁用中断,则忽略本次时钟中断.

内核线程栈,在栈底是一个trapframe,是在从用户线程陷入内核线程的过程中构造的,在栈顶则是一个context结构体,是在进程切换的过程中构造的.

#### 除了`ptable.lock`,为什么禁止在进程切换时持有锁?
答: 在`sched()`中会检查`mycpu()->ncli`,确保不持有其他`spinlock`.

即使我们不考虑`release()`时的校验问题,我们知道自旋锁是轮询始终占用CPU的,假设这样一个场景: 对单核CPU而言,线程A持有锁l在core a上运行,进入了RUNNABLE状态.那么此时如果线程B被调度运行且尝试获取锁l,那么线程B就是一直轮询,浪费了大量的CPU时间.更为严重的是,由于获取锁的过程中,禁用了中断,因此会发生死锁.

### 线程清理
1. `kill(pid)`停止目标进程
2. 因为目标进程可能正在运行或者正持有自旋锁,所以`kill()`并不能清理目标进程所占用的资源,包括内存,文件描述符等等.
3. 我们采用的办法是`kill()`仅对目标进程设置一个标志位`p->killed`,当目标进程在执行`trap()`期间时,如果发现`p->killed`标志位被置位,则自行调动`exit()`退出.
4. 被杀掉的进程会自行调用`exit()`,但是被杀掉的进程是无法自己清理自己的栈空间的,因为正在使用这块栈.
5. 我们采用的办法是将被杀掉的进程标记为`ZOMBIE`状态,然后让被杀掉的进程的父进程来负责清理子进程的内存空间.
6. 将进程状态标记为`ZOMBIE`,那么进程将不会在被执行,而且进程栈也不会再被使用.
7. 父进程将通过`wait()`来执行最终的清理程序.
8. 父进程调用`wait()`后,首先遍历进程列表,发现有子进程为`ZOMBIE`状态,则清理.
9. 若子进程尚未运行完,则会sleep等待被唤醒,执行清理操作.





























---