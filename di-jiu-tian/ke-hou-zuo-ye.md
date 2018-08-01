# xv6 锁实现
## 实验
```
  struct spinlock lk;
  initlock(&lk, "test lock");
  acquire(&lk);
  acquire(&lk);
```
Q. 执行上面代码时,将会发生什么现象?
A. 两次调用`acquire()`时,在第二次调用将会检查,导致走到`panic("acquire")`.线程将会退出.