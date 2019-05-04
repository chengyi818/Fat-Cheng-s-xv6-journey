# Lab6: Network Driver 网络驱动

## Introduction 引言
现在JOS虽然已经有了文件系统,却不没有网络协议栈,任何现代操作系统都不会不支持网络通信.在本实验中,您将为网卡编写一个驱动程序.网卡将基于*Intel 82540EM*芯片,也称为*E1000*.

### Getting Started准备工作
和前几个实验类似,首先执行如下命令以初始化lab6的代码.
```
athena% cd ~/6.828/lab
athena% add git
athena% git commit -am 'my solution to lab5'
nothing to commit (working directory clean)
athena% git pull
Already up-to-date.
athena% git checkout -b lab6 origin/lab6
Branch lab6 set up to track remote branch refs/remotes/origin/lab6.
Switched to a new branch "lab6"
athena% git merge lab5
Merge made by recursive.
 fs/fs.c |   42 +++++++++++++++++++
 1 files changed, 42 insertions(+), 0 deletions(-)
athena%
```

然而,网卡驱动还不足以将JOS连接到互联网.在lab6代码中,已经为我们提供了网络协议栈和网络服务器的代码.与前面的实验一样,使用git获取lab6的代码,合并之前代码,接下来浏览`/net`目录中的文件和`/kern`中的新文件.

除了完成网卡驱动,我们还需要创建一个新的系统调用接口来访问网卡驱动.我们将实现缺失的网络服务器代码,以便在网络协议栈和网卡驱动程序间传输数据包.我们将通过完成一个网络服务器来把所有的代码联系起来.有了新的网络服务器,我们将能够从JOS文件系统中获取文件.

大多数内核设备驱动程序代码都必须从头开始编写.本实验提供的指导比以前的实验少得多:没有框架文件,没有已经写好的系统调用接口,许多设计需要我们自己决策.因此,我们最好在开始单个练习前,先完整阅读整个实验.

---
## QEMU's virtual network  QEMU虚拟网络
我们将使用QEMU的用户模式网络协议栈,因为它在普通用户权限就可以运行.更多用户网络的内容请参考QEMU[文档](https://qemu.weilnetz.de/doc/qemu-doc.html#Using-the-user-mode-network-stack).我们已经更新了makefile以启用QEMU的用户模式网络协议栈和虚拟E1000网卡.

```
     guest (10.0.2.15)  <------>  Firewall/DHCP server <-----> Internet
                           |          (10.0.2.2)
                           |
                           ---->  DNS server (10.0.2.3)
                           |
                           ---->  SMB server (10.0.2.4)
```
默认情况下,QEMU提供一个运行在IP`10.0.2.2`上的虚拟路由器,并将为JOS分配IP地址`10.0.2.15`.为了简单起见,我们在`net/ns.h`将这些默认值硬编码到网络服务器中.

虽然QEMU虚拟网络允许JOS任意连接到互联网,但是JOS的IP地址`10.0.2.15`在QEMU虚拟网络之外没有任何意义(也就是说,QEMU充当了一个NAT),所以我们从外部不能直接连接到JOS内部运行的进程,即使是运行QEMU的主机.为了解决这个问题,我们将QEMU配置为在主机上的某个端口上运行一个服务进程,该服务进程负责连接到JOS中的某个端口,并在真实主机和虚拟网络之间传输数据,即开启端口转发功能.

### Packet Inspection 查看通信报文
makefile还配置了QEMU的网络协议栈,将所有通信数据包记录到lab目录中的qemu.pcap.

要获取捕获数据包的十六进制/ASCII转储,请使用如下tcpdump:
```
tcpdump -XXnr qemu.pcap
```

另外,我们也可以使用[Wireshark](http://www.wireshark.org/)图形界面来查看pcap文件.wireshark支持数百种网络协议,我个人非常喜欢使用.

### Debugging the E1000 调试E1000
我们很幸运能够使用仿真硬件.由于E1000运行在软件中,模拟E1000可以以用户可读的格式向我们报告其内部状态和遇到的任何问题.通常,对于用裸机编写的驱动程序开发人员来说是不可能获取这些信息的.

E1000可以产生很多调试输出,所以我们必须启用特定的日志channel.一些有用的channel如下:
```

| Flag      | Meaning                                            |
|-----------|----------------------------------------------------|
| tx        | Log packet transmit operations                     |
| txerr     | Log transmit ring errors                           |
| rx        | Log changes to RCTL                                |
| rxfilter  | Log filtering of incoming packets                  |
| rxerr     | Log receive ring errors                            |
| unknown   | Log reads and writes of unknown registers          |
| eeprom    | Log reads from the EEPROM                          |
| interrupt | Log interrupts and changes to interrupt registers. |

```

例如,要启用"tx"和"txerr"日志记录,请使用`make E1000_DEBUG=tx,txerr ...`

注意,`E1000_DEBUG`仅针对6.828版本的QEMU有效.

我们可以进一步使用软件模拟硬件进行调试.如果开发过程中陷入困境,不明白E1000为什么没有按照预期的方式响应,可以看看QEMU在`hw/e1000.c`中的E1000实现.

---
## The Network Server 网络服务进程
从零开始写网络堆栈是一项艰苦的工作.相反,我们将使用lwIP,这是一个开源的轻量级TCP/IP协议套件,其中包括一个网络协议栈.你可以在[这里](https://savannah.nongnu.org/projects/lwip/)找到更多关于lwIP的信息.在这个任务中,就我们而言,lwIP是一个黑盒,它实现了一个BSD套接字接口,并且有一个包输入端口和包输出端口.

网络服务进程实际上是四个进程的组合:
1. 核心网络服务进程(包括套接字调用调度程序和lwIP)
2. 输入进程
3. 输出进程
4. 定时器进程

下图显示了不同的进程及其关系.该图显示了包括设备驱动程序在内的整个系统,这将在后面介绍.在本实验中,我们将实现绿色显示的部分.

![lab6_network_server](../images/lab6_network_server.png)

### The Core Network Server Environment 核心网络服务进程
核心网络服务进程由套接字调用调度程序和lwIP两部分组成.套接字调用调度程序的工作方式与文件服务进程完全一样.用户进程使用库函数(在`lib/nsipc.c`中)向核心网络进程发送ipc消息.如果查看`lib/nsipc.c`,我们会发现找到核心网络服务进程的方式与找到文件服务进程的方式相同: `i386_init`用`NS_TYPE_NS`创建了NS进程,所以我们扫描`envs`,寻找这种特殊的进程类型.对于每个用户进程IPC,网络服务进程中的调度程序代表用户调用lwIP提供的BSD套接字接口函数.

普通用户进程不会直接使用`nsipc_*`调用.相反,它们使用`lib/sockets.c`中的函数,后者提供了一个基于文件描述符的套接字应用编程接口.因此,用户进程通过文件描述符引用套接字,就像他们如何引用磁盘上的文件一样.有些操作(connect,accept等)是套接字特有的,但是read,write和close都要经过`lib/fd.c`中的普通文件描述符设备调度代码.就像文件服务进程如何维护所有打开文件的内部唯一标识一样,lwIP也为所有打开的套接字生成唯一标识.在文件服务进程和网络服务进程中,我们使用存储在`struct Fd`中的信息将每个进程的文件描述符映射到这些唯一的标识空间.

尽管文件服务进程和网络服务进程的IPC调度程序看起来是一样的,但还是有一个关键的区别.像accept和recv这样的BSD套接字调用可以无限期地阻塞.如果调度程序让lwIP执行其中一个阻塞调用,调度程序也会阻塞,整个系统一次只能有一个未完成的网络调用.当然这是不可接受的,网络服务进程使用用户级线程来避免阻塞整个服务进程.对于每个传入的IPC消息,调度程序创建一个线程,并在新创建的线程中处理请求.如果线程阻塞,那么只有该线程进入睡眠状态,而其他线程继续运行.

除了核心网络服务进程之外,还有三个辅助进程.除了接受来自用户应用程序的消息,核心网络服务进程的调度程序也接受来自输入和定时器进程的消息.

### The Output Environment 输出辅助进程
当被用户进程套接字调用时,lwIP将生成数据包供网卡传输.LwIP将使用`NSREQ_OUTPUT`IPC发送数据包到输出辅助进程中,数据包附加在IPC消息的页面参数中.输出进程负责接受这些消息,并通过我们即将创建的系统调用接口将数据包转发到设备驱动程序.

### The Input Environment 输入辅助进程
网卡收到的数据包需要注入lwIP中.对于设备驱动程序接收到的每个数据包,输入进程将数据包从内核空间中取出(使用我们即将实现的内核系统调用),并使用`NSREQ_INPUT`IPC消息将数据包发送到核心服务进程.

包输入功能与核心网络进程是分开的,因为JOS使得很难同时接受IPC消息和轮询或等待来自设备驱动程序的包.JOS中没有`select`系统调用,它允许进程监控多个输入源,以识别哪个输入准备好被处理.

如果浏览`net/input.c`和`net/output.c`,会发现两者都有需要实现的部分.这主要是因为实现取决于我们的系统调用接口.实现驱动程序和系统调用接口后,我们将实现这两个辅助进程.

### The Timer Environment 定时器进程
计时器进程定期向核心网络服务进程发送`NSREQ_TIMER`类型的消息,通知它计时器已过期.lwIP使用来自该线程的计时器消息来实现各种网络超时功能.

---
# Part A: Initialization and transmitting packets A: 初始化和传输数据包
JOS内核没有时间的概念,所以我们需要添加它.目前,硬件每10ms生成一个时钟中断.在每个时钟中断,我们可以自增一个变量来表示时间度过了10ms.逻辑在`kern/time.c`中已经实现,但尚未完全集成到JOS内核中.

### Exercise1
为`kern/trap.c`中的每个时钟中断添加一个`time_tick`调用.实现`sys_time_msec`,并将其添加到`kern/syscall.c`中的`syscall`,以便用户空间可以获取时间.

使用`make INIT_CFLAGS=-DTEST_NO_NS run-testtime`,来测试这部分实现.我们应该会看到进程每隔1秒钟从5开始倒数."-DTEST_NO_NS"禁止启动网络服务进程,因为此时进程尚未显示,会导致死机.

## The Network Interface Card 网卡
编写驱动程序需要深入了解硬件及硬件呈现给软件的接口.lab文档将提供如何与E1000交行的上层描述,但在编写驱动程序时,我们还是需要仔细阅读英特尔手册.

### Exercise2
浏览Intel [Software Developer's Manual](https://pdos.csail.mit.edu/6.828/2017/readings/hardware/8254x_GBe_SDM.pdf).手册涵盖几个密切相关的以太网控制器.QEMU模拟`82540EM`.

我们现在应该浏览第二章,了解一下这个设备.要编写驱动程序,我们需要熟悉第3章和第14章,以及4.1(不包括4.1.*).我们还需要参考第13章.其他章节主要涉及E1000的组件,我们的驱动程序不必与之交互.现在不要担心细节;感受一下文档的结构,便于以后查找相关内容.

阅读手册时,请记住E1000是一款具有许多高级功能的复杂设备.正常工作的E1000驱动程序只需要网卡提供的一小部分功能和接口.仔细考虑与网卡交互的最简单方法.强烈建议在利用高级功能之前,首先让基本驱动程序正常工作.

### PCI Interface PCI接口

### Memory-mapped I/O
### DMA
---
## Transmitting Packets
### C Structures
---
## Transmitting Packets: Network Server
---
# Part B: Receiving packets and the web server
## Receiving Packets
## Receiving Packets: Network Server
## The Web Server
