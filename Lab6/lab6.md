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
## QEMU's virtual network
### Packet Inspection
### Debugging the E1000
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
