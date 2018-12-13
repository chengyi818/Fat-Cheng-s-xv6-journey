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
2. 

































































































----