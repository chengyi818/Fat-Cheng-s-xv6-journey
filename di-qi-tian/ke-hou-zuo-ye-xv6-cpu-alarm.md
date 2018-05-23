# xv6 CPU alarm
增加系统调用`alarm`,周期性地回调注册函数.

# 提示
1. 需要修改Makefile,以编译alarmtest.c作为一个xv6用户程序 
2. `user.h`中增加声明`int alarm(int ticks, void (*handler)());`
3. `sys_alarm()`的实现中,需要将alarm间隔和回调函数指针保存到`struct proc`.这里需要修改`proc.h`中的`struct proc`的定义.
4. `sys_alarm()`的实现如下:
```
    int
    sys_alarm(void)
    {
      int ticks;
      void (*handler)();

      if(argint(0, &ticks) < 0)
        return -1;
      if(argptr(1, (char**)&handler, 1) < 0)
        return -1;
      myproc()->alarmticks = ticks;
      myproc()->alarmhandler = handler;
      return 0;
    }
```
5. 需要在`struct proc`中保存距离上次alarm时刻的ticks,或类似的信息.可以`在allocproc()`中初始化这个值.
6. 每次tick,硬件时钟将会触发中断,`trap()`中断号为`T_IRQ0 + IRQ_TIMER`.
7. 如果你只想在进程运行且时间中断来自用户空间时,操作一个进程的alarm ticks.可以加入如下判断`  if(myproc() != 0 && (tf->cs & 3) == 3) ...`.
8. 在`IRQ_TIMER`中,如果alarm计时到了,这时该如何调用用户注册的handler呢?
9. 当调用完用户注册的handler之后,必须继续运行用户程序.这该如何处理?
10. `alarmtest.asm`中有alarm的测试汇编代码.
11. 在单核CPU的情况下运行,会比较有利于gdb调试.`make CPUS=1 qemu`.
12. 当调用handler时,目前不保存调用者的寄存器是没有问题的.

