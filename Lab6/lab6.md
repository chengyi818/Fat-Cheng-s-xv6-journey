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

### Debugging the E1000
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
## The Network Server
### The Core Network Server Environment
### The Output Environment
### The Input Environment
### The Timer Environment
---
# Part A: Initialization and transmitting packets
## The Network Interface Card
### PCI Interface
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
