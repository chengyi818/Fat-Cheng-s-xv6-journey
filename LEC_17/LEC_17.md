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































---
