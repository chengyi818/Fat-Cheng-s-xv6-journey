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


## Q&&A
1. 新的`trap()`实现中有什么安全问题?
2. 为什么不能直接在内核中调用注册函数?
3. 如果回调时,没有保存当前eip,会怎么样?
4. 为什么需要检查CPL为3?
  在内核中处理中断时,依然会产生中断.
5. 如果用户空间注册函数指向了内核地址,会怎么样?
  在从内核返回用户空间后,eip取指时,MMU将会检查权限.


## 中断

### 主题
1. 硬件希望获得关注.
2. 因此,软件不得不放下手头的工作并且回应硬件.

### 陷阱门来源?
1. 硬件中断: 当硬件数据准备完成或者完成一个命令后
2. 异常: 当执行指令过程中,发生page fault或者除0等错误时
3. 系统调用
4. IPI: 内核CPU间通信,比如刷新TLB.

### 硬件中断来源

```
 CPUs, LAPICs, IOAPIC, devices
 data bus
 interrupt bus
```

硬件中断的作用是通知内核发出中断的硬件有事务需要处理,与此相关的是,内核中的驱动则用于处理这些事务.通常硬件中断都是由相应的驱动来进行处理,比如磁盘中断,网卡中断等.但也有一些特殊的硬件中断,比如线程调度,poll等.

### 内核trap()是如何知道中断来自哪个设备?
比如`tf->trapno == T_IRQ0 + IRQ_TIMER`来自哪个设备?

1. 内核通过设置LAPIC和IOPIC来设置中断路由,即硬件中断发往哪个CPU.同时设置了每个中断向量所对应的中断处理函数.比如timer的中断向量号为32.
  
  1.1 `page fault`同样有自己的中断向量号.
  1.2 `LAPIC(local advanced programmable interrupt control)`和`IOAPIC(input output advanced programmable interrupt control)`是计算机硬件的标准组成部分.
  1.3 每个CPU都有一个LAPIC.
  
2. IDT(interrupt descriptor table)是一个地址数组,通过中断向量号在IDT中可以找到对应的中断处理函数地址.
  2.1 IDT的格式是Intel也就是CPU制造商定义的,内核可以修改其中的内容.
  
3. 每个中断向量的处理最终都会到`trapasm.S/alltraps`中.
4. CPU运行中可能发生很多种陷阱,前32号的中断向量都有特定的用途.
5. xv6中系统调用使用的中断向量号为64(0x40)
6. 中断向量号表明了中断的实际来源.

### xv6设置中断向量
1. `lapic.c/lapicinit()`
  * 设置LAPIC硬件如何处理时间中断向量Vector32.

2. `trap.c / tvinit()`
  * 初始化IDT,从代码中我们可以看到中断向量i指向了vector[i]
  * 注意系统调用对应的中断向量被注册为陷阱门而不是中断门.二者的区别在于处理陷阱门时依然接受新的中断,而处理中断门时,则屏蔽了新的中断.

3. 思考题
  * 为什么允许处理系统调用时,接受新的中断?
  * 为什么处理设备中断时,禁止接受新的中断?
  
4. `Vector.S`
  * 通过`vectors.pl`自动生成
  * 首先,push一个错误码返回值空间,用于从中断返回时保存错误码.
  * 然后,push一个中断向量号.在中断处理时,通过`tf->trapno`可以读取.
  
### 中断处理时使用的内核栈
1. 用户空间切换到内核的时间点?
  * 系统调用处理INT指令
  * 设备中断处理时
  
2. 硬件提供了TSS(task state segment),以便让内核可以配置CPU
  * 每个CPU均有对应的TSS
  * 每个CPU均可以运行不同的进程,并在不同的内核栈上处理中断
  
3. `proc.c/scheduler()`
  * 每个CPU独立调用,用于寻找合适的进程供CPU运行.
  * 其中将会调用`switchuvm()`

4. `vm.c/switchuvm()`
  * 根据将要运行的proc, 设置CPU使用的内核栈
  * 根据将要运行的proc, 设置内核使用的page table
  
### 思考题
1. 当CPU处理中断陷入内核时,保存的eip应该指向哪里?
  * 当前正在执行的指令?
    如果执行指令时,发生page fault且启用了lazy allocation.那么应该重新执行该指令.
  * 下一条待执行的指令
    如果是一次正常的系统调用,那么应该执行下一条指令.

## 额外设计文档
1. 中断曾经是比较快的一种方案.现在中断是一种比较慢的方案.
  * 旧方案: 每个事件都会触发中断,简单的硬件,复杂的软件.
  * 新方案: 硬件完成大量的工作,避免发生中断.
  
2. 中断处理的损耗为微秒级
  * 保存/恢复状态
  * 缓存未命中,导致缓存更新.
  
3. 一些硬件设备每微秒产生不止一次中断.
  * 一个GB级的以太网卡每秒可以产生150w次中断
  
4. 在处理高速设备时,使用轮询的方式而不是中断
  
5. 处理慢速设备时,采用中断的方式
  * 对于键盘这样的设备,持续的轮询将会浪费CPU资源.
  
6. 在轮询和中断间自动切换
  * 根据需要处理的频率来自动切换模式
  * 高速时,使用轮询的方式
  * 低速时,使用中断的方式
  
7. 更快地转发中断到用户空间
  * page fault 和 用户相关的设备(TODO)
  * 硬件直接传递给用户空间,减少内核的介入.
  * 通过内核更快地传递路径,从而实现DMA.






























----

