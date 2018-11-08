# 课前准备
1. 阅读**第五章 进程调度**,直到*sleep and wake up*
2. 阅读`xv6`代码中的`proc.c`和`swtch.S`

# 摘要
1. 进程调度
```
yield() --> sched() --> swtch() --> scheduler() --> swtch()
```

2. sleep, wakeup实现

3. 管道pipe实现

4. wait, exit, kill实现