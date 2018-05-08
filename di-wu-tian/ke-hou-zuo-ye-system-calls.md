# Homework: xv6 system calls
本作业旨在增加对系统调用的理解.

## 打印系统调用的函数名和返回值
在跟踪完系统调用的流程后,我们会发现这个并不十分困难.主要修改在`syscall.c`文件的`syscall()`函数中.

## 增加一个新的系统调用date
通过这个练习,我们可以从头到尾走一遍系统调用的流程.依次为:
1. 用户空间date声明
2. 用户空间date实现
3. 增加内核date系统调用号
4. 内核date具体实现sys_date
5. 向内核系统调用表增加date的系统调用号和内核实现.

注:
这里还有一个点是关于如何从用户空间拿到用户传入的参数.主要是通过用户空间%esp的相对位置获取的.