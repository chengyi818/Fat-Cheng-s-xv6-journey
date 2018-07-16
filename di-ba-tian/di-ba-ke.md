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


### sys_alarm 处理
1. 内核处理系统调用时,是通过trapframe->eax寄存器中的内容,来判断用户空间调用的具体系统调用.
2. `sys_alarm`是通过访问用户空间栈来获取系统调用所需的参数内容.

### 处理时钟中断
1. 检查剩余时间
2. 如果超时,则回调注册函数.
3. 重置剩余时间.
4. 时钟中断号`IRQ_TIMER`.

### 回调注册函数
1. 在内核中,通过操作用户空间函数栈的方式来调用注册函数.
2. 将esp指针下移,将当前eip保存.
3. 将eip指向注册函数.
