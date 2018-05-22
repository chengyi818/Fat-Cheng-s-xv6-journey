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
6. 每次tick,硬件时钟将会触发中断,`trap()`中断处理为`T_IRQ0 + IRQ_TIMER`.
7. 