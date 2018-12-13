# 用户态线程
本章节我们将通过实现代码来执行线程之间的上下文切换,从而完成一个简单的用户级线程包.
[相关链接](https://pdos.csail.mit.edu/6.828/2017/homework/xv6-uthread.html)

## 线程切换

### 下载编译uthread相关代码
1. 下载[uthread.c]()和[uthread_switch.S]()到xv6目录,确保`uthread_switch.S`后缀为`.S`.
2. 将如下规则添加到xv6 Makefile `_forktest`之后,Makefile命令起始不是空格,而是tab.
```
_uthread: uthread.o uthread_switch.o
		$(LD) $(LDFLAGS) -N -e main -Ttext 0 -o _uthread uthread.o uthread_switch.o $(ULIB)
		$(OBJDUMP) -S _uthread > uthread.asm
```

3. 将`_uthread`添加到xv6 Makefile中的`UPROGS`.
4. 运行xv6,在xv6 shell中执行`uthread`.xv6 内核将会打印出`page fault`.

### 目标
1. 我们的目标是通过完成`thread_switch.S`,来使得在单核情况下,打印信息如下:
```
~/classes/6828/xv6$ make CPUS=1 qemu-nox
dd if=/dev/zero of=xv6.img count=10000
10000+0 records in
10000+0 records out
5120000 bytes transferred in 0.037167 secs (137756344 bytes/sec)
dd if=bootblock of=xv6.img conv=notrunc
1+0 records in
1+0 records out
512 bytes transferred in 0.000026 secs (19701685 bytes/sec)
dd if=kernel of=xv6.img seek=1 conv=notrunc
307+1 records in
307+1 records out
157319 bytes transferred in 0.003590 secs (43820143 bytes/sec)
qemu -nographic -hdb fs.img xv6.img -smp 1 -m 512 
Could not open option rom 'sgabios.bin': No such file or directory
xv6...
cpu0: starting
init: starting sh
$ uthread
my thread running
my thread 0x2A30
my thread running
my thread 0x4A40
my thread 0x2A30
my thread 0x4A40
my thread 0x2A30
my thread 0x4A40
....
```

### 分析
1. `uthread.c`创建了两个线程,并在他们之间来回切换.每个线程都会打印`my thtread……`,然后让出CPU给其他线程运行机会.
2. 为了观察到上述现象,我们需要完成`uthread_switch.S`.当然在此之前,我们还需要了解`uthread.c`是如何使用`uthread_switch`的.
3. `uthread.c`中,有两个重要的全局变量: `current_thread`和`next_thread`.它们各自指向一个`thread`结构体.`thread`结构体从高到底,依次为线程状态,线程栈和栈顶指针保存区.
4. `uthread_switch`的任务是:
  * 保存当前线程状态到`current_thread`.
  * 从`next_thread`中恢复新的线程状态
  * 将`current_thread`指向`next_thread`所指向的位置.
  * 这样当`uthread_switch`返回后,将执行`next_thread`,且为`current_thread`.
5. `thread_create`创建了一个新的线程,它提供了`uthread_switch`应该如何实现的提示.
6. `uthread_switch`使用`pushal`和`popal`来一次性push或restore全部的8个寄存器.
7. `uthread_create`模拟了全部8个寄存器的值,即32个字节,全部为0.
8. 如下是`struct thread`在内存中的布局:
```
    --------------------
    | 4 bytes for state|
    --------------------
    | stack size bytes |
    | for stack        |
    --------------------
    | 4 bytes for sp   |
    --------------------  <--- current_thread
         ......

         ......
    --------------------
    | 4 bytes for state|
    --------------------
    | stack size bytes |
    | for stack        |
    --------------------
    | 4 bytes for sp   |
    --------------------  <--- next_thread
```
9. `current_thread`和`next_thread`即指向`struct thread`的首地址.
10. 如果我们希望修改`current_thread`指向的`struct thread`中的`sp`的值,汇编代码如下:
```
   movl current_thread, %eax
   movl %esp, (%eax)
```
11. 以上汇编代码会将%esp寄存器中的值赋给`current_thread->sp`,这是因为`sp`的起始地址就是`current_thread`,还记得上面`struct thread`所展示的内存布局图么?
12. 在编译完成后,我们可以通过`uthread.asm`来学习相关的汇编代码.
13. 为了开发调试,我们还可以单步调试`uthread_switch`,以下是操作方法:
```
(gdb) symbol-file _uthread
Load new symbol table from "/Users/kaashoek/classes/6828/xv6/_uthread"? (y or n) y
Reading symbols from /Users/kaashoek/classes/6828/xv6/_uthread...done.
(gdb) b thread_switch
Breakpoint 1 at 0x204: file uthread_switch.S, line 9.
(gdb) 
```
14. 以上添加的断点,可能在运行`uthread`之前就被触发,请问是为什么?
15. 当我们添加断点之后,运行`uthread`,gdb将会在`uthread_switch`处停止,我们可以通过如下方法查看`uthread`的状态.
```
(gdb) p/x next_thread->sp
$4 = 0x4ae8
(gdb) x/9x next_thread->sp
0x4ae8 :      0x00000000      0x00000000      0x00000000      0x00000000
0x4af8 :      0x00000000      0x00000000      0x00000000      0x00000000
0x4b08 :      0x000000d8
```
16. 






























































































----