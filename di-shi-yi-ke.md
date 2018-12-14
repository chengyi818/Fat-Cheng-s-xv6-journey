# 第十一课 进程同步

# 内容大纲
1. 内容收尾: 进程调度
2. 课后作业点评: 用户线程切换
3. 进程同步:
    * xv6的sleep和wakeup机制
    * wakeup丢失问题
    * 终止
    
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

## 