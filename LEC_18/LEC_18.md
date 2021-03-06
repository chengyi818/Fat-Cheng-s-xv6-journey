# LEC18 Scalable Locks

## 本论文的目的
1. 论文中图2展示的场景在多核场景下简直是场灾难.
2. 锁导致性能大幅度下降,增加更多的核并没有能够提升性能.
3. 这种`non-scalable lock`现象非常重要.
4. 现象出现的原因值得探究和思考.
5. 这些解决方案是并行编程中的巧妙练习

## 核心问题
1. 问题为: 多核场景中cache缓存间,锁信息的交互.

## 多核基本场景
1. 我们已经有了基本的多核场景的模型概念.
2. 多个核之间共享总线和内存.
3. 为了实现锁获取的功能,x86的`xchg`指令会锁住总线,从而为`xchg`提供原子性.

## 实际场景
当然实际场景要复杂的多.对于CPU而言,总线和内存都是非常慢的.因此每个CPU都会拥有自己的cache缓存.

如果CPU要访问的数据位于cache中,那么只需要几个指令周期就可以完成数据读取,如果cache未命中,从内存中读取数据,往往需要100个指令周期以上.

## cache同步
既然每个CPU都有自己的cache缓存,那么如何保证每个CPU的cache的内容是正确的呢?

举个例子:
1. core 1读取变量x的值为10.
2. core 2将x的值修改为11.
3. 此时,core 1读取变量x的值为多少呢?

此时,我们需要引入缓存一致性协议(cache coherence protocal).从而保证每个CPU都可以读取到最新的写入内容.

## 缓存一致性如何运行?
1. 有许多方案,下面是个简单实现.
2. 每条cache line都有自己的状态, 地址和64字节的数据.
3. 状态包括以下三种: 已修改(Modified), 共享(Shared), 失效(Invalid)
4. 当CPU读写数据时,将会在核间交换信息.

## 简化的消息类型
1. invalidate(addr): 使得cache line失效
2. find(addr): addr上的数据是否在其他核上有拷贝.
3. 所有消息都会广播给所有核.

## 下面展示了核间如何同步?
```
  I + local read -> find, S
  I + local write -> find, inval, M

  S + local read -> S
  S + local write -> inval, M

  S + recv inval -> I
  S + recv find  -> nothing, S

  M + recv inval -> I
  M + recv find  -> reply, S
```

1. 如果已经是Shared,那么读取可以不需要bus通信.
2. 如果已经是Modified,那么写入可以不需要bus通信.

## 两个核之间可能的状态
```
		  core1
		  M S I
		M - - +
core2   S - + +
		I + + +
```

1. 对于每个cache line,最多一个核处于Modified.
2. 对于每个cache line,要么一个核为Modified,要么均为Shared,没有其他情况.

Q: 什么样的使用模式受益于这种一致性方案？

A:

1. 只读数据(每个cache都有一份数据copy)
2. 仅有一个core反复写数据(Modified提供了排他性的写入)

还有其他的可能方案,比如写入会更新其他拷贝的内容,但是失效似乎更好.

## 真实方案
1. 真实硬件使用的方案更加巧妙
2. 链路网络取代了bus总线,单播取代广播.
3. 使用分布式目录,用于跟踪哪些内核缓存每一行
4. 单播查找到目录

Q: 如果我们有了cache一致性,为什么还需要锁?
A: cache一致性保证了CPU可以读取到最新的数据,而锁避免了在读取-修改-写回循环中不丢失更新,同时避免其他人读取到操作未完成的数据.

## 开发者根据硬件提供的原子指令构建了锁
1. xv6使用了原子性的交换
2. 其他锁的实现可能利用了`test-and-set`,或者原子自增等.
3. 类似`__sync__`等函数,最终将转化为原子指令

## 硬件是如何实现原子指令的?
1. 当cache line设为Modified.
2. 推迟处理所有一致性消息
3. 完成操作(读取旧值,写入新值)
4. 恢复处理一致性消息

## 锁的性能
1. 假设同时有N个核在等待锁
2. 从前一个持有者到后一个持有者,lock额外损耗的时间是多少?
3. 性能瓶颈通常存在于核间内部通信.所以我们将根据msg的数量来衡量成本.

## 我们期待怎样的性能?
1. 假设N个核在等待.
2. 我们希望所有核总等待时间复杂度为O(N)
3. 我们希望每个临界区和损耗时间复杂度为O(1**,即和参与的核数无关.

## 测试并设置xv6 spinlock
等待CPU将会重复执行*原子交换*操作,这会有什么问题么?
当然,我们并不关心等待CPU本身所消耗的时间,因为它本来就在等待执行.我们关心的重点在于等待CPU是否降低了锁持有CPU的执行速度.

临界区和释放锁的执行时间,锁持有CPU必须等待以使用总线,所以锁持有CPU的内存操作时间复杂度为O(N),所以损耗时间复杂度O(N).

## O(N)的损耗是一个严重的问题么?
当然,我们希望损耗时间复杂度为O(1).O(N)的损耗时间意味着所有核的总计损耗时间为O(N^2),而不是O(N).

## linux ticket lock
### ticket lock的设计目标:
1. 只读自旋锁,而不是不断地重复原子交换指令.
2. 公平(相较而言,test&set锁是不公平的**

### ticket lock设计思想
在申请ticket lock时,为每个CPU分配一个数字,并依次唤醒持有这些数字的进程.从而避免了每个等待进程都在不断地执行*原子交换*.

### 思考
1. ticket lock为什么性能优于t-s lock?
减少了core inherence message的交互.
2. 为什么ticket lock是公平的?
每个进程在刚申请锁时,即分配到了相应的数字.

### 时间分析

#### 获取锁时
1. 原子递增,并广播msg.仅发生一次,不会重复执行.
2. 只读自旋,没有其他损耗,直到发生release.

#### 释放锁时
1. 发送`invalidate`msg
2. N个CPU发送`find`msg,更新锁的值.
3. 时间复杂度为O(N).

综上,时间复杂度和`test&set`是一致的.


## non-scalable lock
1. test&set lock和ticket lock都是non-scalable lock.
2. 其特征是,时间复杂度为O(N).

## non-scalable lock会带来严重的问题么?
毕竟,相较于锁而言,程序还做了很多其他的工作.或许锁的损耗可以忽略不计呢.

## 观察paper的Figure 2
首先,让我们观察Figure 2(c),PFIND.其中x轴代表CPU的数目,y轴每秒`find`完成的数目(总吞吐量).

1. 为什么吞吐量上升?
2. 为什么吞吐量不变?
3. 为什么吞吐量下降?
4. 是什么在决定最大吞吐量?
5. 为什么下降地如此迅速?

## 突然下降的原因
1. 根据Figure 3最后一列,我们看到临界区占每个核的7%左右.
2. 在14个core的情况下,大概会有1到2个core在临界区中.
3. 所以看起来很奇怪为什么性能下降地如此迅速.

然而:
1. 一旦两个core开始争抢锁,临界区+额外损耗会开始增长.
2. 所以真实占用时间会超过7%.
3. 所以进一步导致更多的core在等待.
4. 随着core的增多,拥塞会更加严重.

## 另一个角度分析
```
  acquire(l)
  x++
  release(l)
```

1. 临界区如此之短,是否不会对总体性能造成影响?
2. 如果仍是本core获取锁,仅会需要几十次指令周期.因为所有操作都在cache中完成.
3. 如果是其他core之前占用锁,那么可能需要100个指令周期.
4. 如果许多core发生了竞争,那么可能需要1w个指令周期.
5. 很多内核操作也仅会消耗100个指令周期,所以一个竞争的锁会导致消耗的指令周期增加100倍.

## 如何使锁有良好的伸缩性?
1. 当释放锁时,仅发送O(1)的msg.
2. 当释放锁时,保证只有一个core来读写锁
3. 每次仅唤醒一个core.

## 思路
1. 如果每个core在不同的cache上自旋会怎样？
2. 此时的抢锁消耗为: 原子自增,然后只读自旋.
3. 释放消耗为: `invalidate`下一个占有锁的slot,只有它需要重新load,不涉及其他core.
4. 所以每次释放的时间复杂度为O(1).
5. 缺点在于: 比较大的内存消耗.每个锁都需要N个slot.通常比受保护对象的还要大得多.
6. 本思想由Anderson提出

## MCS
1. [code](https://pdos.csail.mit.edu/6.828/2017/lec/scalable-lock-code.c)
2. 目标: 效果类似Anderson,但是减少内存使用.
3. 思路: 为每个lock创建一个等待链表.
4. 思路: 每个线程都是链表中的一个节点,因为每个线程只能等待一个lock,所以总的内存消耗为O(locks+threads),而不是O(locks*threads).
5. `acquire()`将调用者插入等待链表尾部,然后调用者在自己的节点上自旋.
6. `release()`唤醒下一个节点,同时删除本节点.
7. API需要稍微修改下(需要传入qnode来获取和释放锁)


## scalable lock的性能
1. Paper中的Figure10展示了ticket lock,MCS lock及优化后的对比结果.
2. 其中,x轴表示核数,y轴表示总吞吐量.

Q: 为什么总吞吐量没有随着核数的增加而增长?
1. ticket lock在两个核时性能最佳,只有一个原子指令.
2. ticket lock伸缩性不佳,消耗随着核数增加而增加.
3. MCS伸缩性很好,消耗随着核数增加而保持不变.

## Figure 11
1. 展示了没有竞争时的耗时.在没有竞争时,非常快速.
2. ticket lock在获取锁时,使用了一个原子指令,所以耗时是释放的10倍.
3. 如果另外一个core之前占有锁,则耗时会进一步上升.

## scalable lock使内核伸缩性变好了?
1. 没有,伸缩性受限于临界区的大小,scalable lock避免了性能大幅下降.
2. 为了解决伸缩性的问题,需要重新设计内核子系统.

---

## Linux内核和MCS locks
Linux内核有可伸缩(或不会性能急剧下降)的锁,并且正在使用它.修复基于ticket的自旋锁的性能的最早努力可以追溯到2013年的[1],似乎使用了我们今天读到的这篇论文.(但是那个特别的补丁实际上并没有合入内核主线).大约在同一时间,用MCS锁实现了互斥锁[2].

然而,用可伸缩的锁替换ticket自旋锁却是一个很大的挑战.更难的问题在于,像MCS这样的锁定方案会使每个自旋锁的大小膨胀,这是不希望的,因为自旋锁在Linux内核中有额外的大小限制.

几年后,Linux开发人员想出了`qspinlocks`(使用MCS机制,用巧妙的技巧来避免自旋锁的大小膨胀)替换ticket自旋锁,它现在是自2015年以来Linux内核[3][4]中默认的自旋锁实现.你也可以找到这篇[5]文章(Linux锁子系统的贡献者之一写的),非常有趣.

旧的不再使用的的ticket自旋锁实现已在2016年[6]从代码库中删除.

[1]. Fixing ticket-spinlocks:
https://lwn.net/Articles/531254/
https://lwn.net/Articles/530458/

[2]. MCS locks used in the mutex-lock implementation:
http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=2bd2c92cf07cc4a

[3]. MCS locks and qspinlocks:
https://lwn.net/Articles/590243/

[4]. qspinlock (using MCS underneath) as the default spin-lock implementation:
http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=a33fda35e3a765

[5]. Article providing context and motivation for various locking schemes:
http://queue.acm.org/detail.cfm?id=2698990

[6]. Removal of unused ticket-spinlock code from the Linux kernel:
http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=cfd8983f03c7b2


















---
