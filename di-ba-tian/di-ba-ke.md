# 第八课 系统调用, 中断, 异常

## 第七课 课后作业
[添加CPU alarm](https://pdos.csail.mit.edu/6.828/2017/homework/xv6-alarm.html)

##提示
有三个注意点:
1. 添加一个新的系统调用
2. 通过时间中断,来计算程序已经运行的时间.
3. 内核回调注册函数.

### 添加系统调用
```
  syscall.h: #define SYS_alarm 22
  usys.S: SYSCALL(alarm)
    alarmtest.asm -- mov $0x16,%eax -- 0x16 is SYS_alarm
  syscall.c syscalls[] table
  sysproc.c sys_alarm()
```
注意这里,由于我们之前添加了date系统调用,因此我使用的系统调用号为23.

本质上, `alarm`是希望实现从用户空间`alarm()`函数到内核`sys_alarm()`函数的调用.由于进程隔离的需要,因此我们必须使用一种间接的调用方法.也就是中断的方式.



