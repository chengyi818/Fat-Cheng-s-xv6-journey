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
1. guest使用硬件标志位
  1.1 当CPL=0时,pushf/popf等指令会读写该标志位
  1.2 从guest的角度,硬件运行正常
2. 当发生硬件设备中断
  2.1 通过VMM配置VMCS(virtual machine control structure),来控制每个中断是由guest还是由host响应.
  2.2 如果由guest响应:
    2.2.1 硬件首先检查guest的可中断标志位
    2.2.2 通过guest的IDT获取硬件中断向量
    2.2.3 不需要退出guest状态
  2.3 如果由host响应
    2.3.1 VT-x退出guest状态,返回VMM.VMM处理中断.

## VT-x: 页表
1. EPT: 第二层地址转换
2. EPT由VMM控制
3. %cr3寄存器由guest控制
4. guest虚拟地址通过cr3转换为guest物理地址,guest物理地址通过EPT转换为host物理地址.
  4.1 cr3寄存器保存的是guest物理地址
  4.2 EPT保存的是host物理地址.
5. EPT对于guest而言,是不可见的.
6. 所以:
  6.1 guest可以自由读写cr3寄存器,修改PTE.
  6.2 VMM通过EPT提供隔离
7. 典型设置:
  7.1 VMM分配size大小的物理内存供guest使用
  7.2 VMM将guest物理内存地址,从0到size映射到EPT中.
  7.3 guest使用cr3来配置guest的进程地址空间.















---

# Dune
