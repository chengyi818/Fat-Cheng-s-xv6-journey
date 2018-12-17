# 第十一课 进程同步

# 内容大纲
1. 内容收尾: 进程调度
2. 课后作业点评: 用户线程切换
3. 顺序协同:
    * xv6的sleep和wakeup机制
    * wakeup丢失问题
    * sleep期间,进程终止
    
# 进程调度    
## 进程切换示意图:
1. 下图显示了一个内核线程从`yield()`到`scheduler()`线程的过程:
```
    yield       scheduler
    acquire     release
    RUNNABLE
    swtch       swtch
      |           ^
      |           |
      -------------
```

## ptable.lock的使用
1.  xv6进程切换中,获取ptable.lock和释放并不在同一个线程中,这种用法通常比较罕见.
2. 在xv6中,大部分锁的使用都是在同一线程中.
3. 线程切换和锁的这种用法在OS的实现中并不罕见.

## xv6关于上下文切换和并发的问题
参见第十课的内容

---

## 课后作业: 用户线程上下文切换

### gdb调试
```
  (gdb) symbol-file _uthread
  (gdb) b thread_switch
  (gdb) c
  uthread
  (gdb) p/x next_thread->sp
  (gdb) x/9x next_thread->sp
   (gdb) p/x &mythread
```

### 栈上第九项的值代表什么含义?
答: 存放的是线程的可执行代码地址,该地址会被载入到CPU的eip寄存器中.

### 为什么需要将next_thread拷贝到current_thread?
答: current_thread标识了当前正在运行的线程,当线程切换完成,需要修改current_thread的值,以保证下次thread_schedule运行正常.

### 为什么uthread_yield只会在用户空间调度,而不是调用到内核?
答: 因为该函数仅存在于用户空间,并没有使用到系统调用.因此不会陷入内核.

### 当uthread因系统调用阻塞时,会发生什么?
答: uthread在内核中仅存在一个线程,是在用户空间模拟了多个用户线程.因此当uthread陷入内核时,用户空间其他模拟线程也不会被调度执行.

### uthread可以利用多核CPU并发执行么?
答: 不能,因为uthread是在一个核上模拟了用户空间多线程,自始至终仅有一个核在运行.这些用户空间线程不会被同时调度到不同的核上运行.

---

# 顺序协同

## 进程在执行过程中,可能因为需要等待某些事件,比如:
* 等待disk io完成
* 等待pipe reader后读取一部分pipe内容后,以便有空间可以写入
* 等待子进程退出,以便可以及时清理子进程的资源

## 在等待特定事件时,是否可以通过while-loop轮询的方式呢?
答: 尽量不要,因为我们不能判断等待时长.轮询是对CPU资源的一种浪费.

## 更好的做法: 协同使用CPU
1. xv6的sleep和wakeup机制
2. 条件变量
3. 内存屏障
4. ...

## sleep && wakeup

### sleep(chan, lock)
进程将睡眠在`chan`上,其中`chan`是进程等待的条件的地址.

### wakeup(chan)
将唤醒所在等待在`chan`上的进程,可能唤醒不止一个进程.

### 注意
这里睡眠和唤醒仅仅是一个大概率的可能,并没有正式的规定说 睡眠进程必须在等待条件发生时才会被唤醒.事实上即使等待条件尚未发生,睡眠进程也可能被唤醒.因此睡眠进程在使用`sleep()`时,应当再次确认等待事件是否发生.

## sleep/wakeup机制面临的问题
1. wakeup丢失
2. 睡眠过程中,进程中止

## sleep/wakeup示例: iderw()/ideintr()
我们以ide硬盘读取为例:
1. 首先`iderw()`将块b读取请求放入请求队列,然后`sleep`在块b上.
2. 块b是一个缓冲区,从ide硬盘读入的内容,将会被存放在此.
3. 执行`iderw()`的线程将会进入睡眠,等待硬盘读取操作完成.当读取完成后,块b的B_VALID标志位将被置位.
4. 当ide读取完成后,硬件将会发出中断,进入ide中断处理,执行`ideintr()`
5. `ideintr()`主要进行: 将块b标识为B_VALID,然后唤醒所有等待块b的进程.

## 思考点
`iderw()`在睡眠时,将会持有`idelock`.同时`ideintr()`也需要持有`idelock`,那么为什么`iderw()`不在调用`sleep()`前,就释放`idelock`呢?
答: 这里主要是为了针对`wakeup`丢失的问题.假设这样一个场景: 线程A执行`iderw()`发起了一次ide请求,然后释放了`idelock`.此时线程A正准备执行`sleep`,如果此时ide请求已经完成,那么中断处理程序就会执行`ideintr()`,尝试唤醒线程A.刚被唤醒的线程A立刻就会执行`sleep()`,而到了这会就没有中断回来唤醒线程A,这样线程A在没有`sleep`时,就收到了`wakeup`.在真正睡眠后,就无法再被调度执行.

## 如何解决wakeup丢失的问题?



























































---
