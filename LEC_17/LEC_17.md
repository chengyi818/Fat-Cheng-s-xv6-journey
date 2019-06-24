# LEC17 Singularity

# 预先阅读
1. [Singularity](https://github.com/chengyi818/my_xv6/blob/master/resources/LEC17/hunt07singularity.pdf)
2. [Language Support for Fast and Reliable Message-based Communication in Singularity OS](https://github.com/chengyi818/my_xv6/blob/master/resources/LEC17/singularity-eurosys2006.pdf)

---

# 概览

## Singularity是微软的研究型实验OS
1. 该OS发表了很多相关论文,相当高调.
2. 受到了微软OS的经验的影响,比如Windows
3. 我们可以推测Singularity对于微软产品的影响

## 目标
1. 健壮性和安全性增强,尤其针对插件
2. 减少不必要的交互
3. 利用最新的技术

## 总体架构
1. 微内核: 内核,进程和IPC
2. `page 5`声称已经将服务分解到用户流程中
  2.1 NIC, TCP/IP, FS, disk driver
  2.2 内核: 进程, 内存, 部分IPC, nameserver
  2.3 不再追求和UNIX的兼容,因此避免了部分陷阱
3. 最终,`singularity`有192个系统调用

## 最根本的设计理念
1. 只有一个地址空间(不需要分段和分页功能),内核和所有进程均处于同一地址空间
2. 用户进程均运行在特权级,即CPL=0

## 新设计的优势
1. 性能提升
2. 更快的进程切换: 不需要切换页表
3. 更快的系统调用: 不需要`INT`就可以执行系统调用
4. 更快的进程间通信: 不需要拷贝数据
5. 用户进程可以直接访问硬件资源,比如设备驱动
6. paper Table 1展示了性能对比的结果

## 新设计的核心目标
1. 健壮性
2. 安全性
3. 交互

## 健壮性不依赖页表保护机制
1. 不可靠的问题主要来自浏览器插件以及动态加载的内核模块
2. 出于性能和便利性的考量,需要加载到主进程的地址空间来执行.
3. 我们是否可以不依赖硬件,来完成健壮性保护

## 插件在Singularity中如何运行?
1. 插件包含: 设备驱动,新的网络协议,浏览器插件
2. 分隔的进程,和主进程间通过IPC通信

## 单地址空间遇到的挑战
1. 阻止恶意程序访问其他进程或者内核空间
2. 支持杀死进程和进程退出


---

# SIP: software-isolated program

## SIP总体理念
1. 密封的
2. 进程外部无法修改程序
  2.1 除了启动和停止进程外,没有以进程号为参数的系统调用
  2.2 可能没有debugger,只有IPC.
3. 进程内部无法修改程序
  3.1 没有JIT
  3.2 没有class loader
  3.3 没有动态库加载

## SIP规则
1. 指针只能指向本进程的数据
  1.1 指针不允许指向其他SIP数据或者内核
  1.2 尽管共享地址空间,但没有共享内存.
  1.3 在exchange heap中传递的IPC消息,只支持有限的exception
2. SIP可以从内核中分配物理内存
  2.1 不同的分配内存是不连续的.

## 为什么要限制SIP的修改?即使是修改本进程?
1. 限制的好处是什么?
  1.1 没有代码注入攻击
  1.2 更容易完成正确性推理
  1.3 更容易做代码优化,比如删除未使用的函数
  1.4 TODO: SIP可以作为一个安全原则,拥有文件
2. 以上收益和损失相比,是否值得?

## 为什么不类似Java虚拟机,共享所有数据呢?
1. SIPs排除了所有的进程间交互,除了显式的通过IPC通信
2. SIPs更加健壮
3. SIPs使得每个进程都有自己的语言runtime,GC等
  3.1 尽管质量有一定保障,但这部分代码未必没有bug
  3.2 内核代码也是同样敏感
  3.3 所以开发者比较难自己准备runtime和GC
4. SIPs使得内核杀死和退出进程比较容易


























---
