# Homework

Q: ticket锁与JOS/xv6中的spinlock略有不同.假设基于本文中描述的失效一致性方案,ticket锁引入的CPU间通信比JOS/xv6自旋锁少吗?

A: 少,在ticket lock中,等待线程不断比较index和自身获取的ticket是否相等,index值是只读的,只需要等待持有lock的线程更新.
在xv6 spinlock中,每次xchg都会改变lock的值,从而invadate持有线程的cache.
