# 第九课 锁

## 为什么要需要锁?
1. 应用程序需要同时使用多个物理核来加速运行
2. 因此内核必须支持并发调用系统调用
3. 同时,可能出现并发地访问内核数据(buffer cache, processes)
4. 锁帮助了正确地实现数据共享
5. 但是锁的引入,也导致性能有一定的损失

## 第八天 Homework: 锁

## 锁
 
```
  lock l
  acquire(l)
    x = x + 1 -- "critical section"
  release(l)
```


