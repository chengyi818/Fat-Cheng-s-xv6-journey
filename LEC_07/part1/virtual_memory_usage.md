# 使用虚拟内存

# 进程隔离
1. 用户进程不能执行特权指令
2. 用户进程只能通过系统调用进入内核
3. 只有内核才能切换%cr3 页表寄存器.
4. 内核页表没有U标志位,因此用户不能使用内核页表转换虚拟地址.
5. 内核可以读写用户内存.
6. 内核执行系统调用时,需要检查用户传入的参数.

# xv6 虚拟内存布局
1. 内核为什么需要使用虚拟内存?