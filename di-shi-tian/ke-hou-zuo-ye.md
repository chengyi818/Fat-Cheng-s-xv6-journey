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

3. 

































































































----