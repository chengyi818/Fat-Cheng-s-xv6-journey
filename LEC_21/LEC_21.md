# LEC21: 虚拟机Dune

## 大纲
1. 虚拟机
2. x86虚拟机制和VT-x
3. Dune

---

# 虚拟机

## 什么是虚拟机?
模拟计算机,以准确运行一个OS

## 图示
参加paper

## 为何讨论虚拟机?
1. VMM也是某种类型的内核,包含进程调度,进程隔离,资源分配
2. 当前的做法是VMM和guest OS协同工作
3. 是否可以通过虚拟机来解决某些OS难题

## 虚拟机的作用
1. 在一台物理主机上运行很多guest OS.
  1.1 对于云计算企业而言,经常需要为每个服务提供一个独立隔离的运行环境并分配一定的资源.
  1.2 如果每个服务都运行在一台独立的物理主机上,将会造成巨大的资源浪费.
2. 虚拟机可以提供比进程更好的隔离效果.
3. 在一台物理主机上运行不同的操作系统.
4. 提供内核开发环境,比如qemu

---

# 基于硬件的虚拟化

## 硬件虚拟化机制
1. Intel VT-x, VMX, AMD SVM
2. 虚拟机概念的火热推动了Intel和AMD增加了硬件对虚拟化的支持.
3. 使得实现VMM(virtual machine monitor)变得更加容易.

## VT-x
1. 增加了两种模式: root和non-root.
2. VMM运行在VT-x的root模式.
  2.1 VMM可以修改VT-x的控制数据结构,比如VMCS和EPT
3. guest运行在non-root模式
  3.1 guest在CPL=0的特权级下,可以直接使用硬件
  3.2 此时VT-x会检查guest的某些操作,并使guest返回VMM.
4. 为了支持新的两种模式,增加了一些指令.
  4.1 VMLAUNCH/VMRESUME: root -> non-root
  4.2 VMCALL: non-root -> root
  4.3 另外,某些中断和异常也会导致guest返回VMM
5. 虚拟机控制数据结构(VM control structure,VMCS)
  5.1 包含了在两种模式间转换所需保存和恢复的状态.
  5.2 配置信息,比如是否需要在缺页异常时陷入root模式.

## pushf/popf 示例















---

# Dune
