# shell
本作业主要是通过实现几个Feature来熟悉Unix系统调用接口和shell.

# 准备工作
## 阅读
阅读xv6 book的第0章

## shell源码
下载[sh.c](https://pdos.csail.mit.edu/6.828/2017/homework/sh.c),并仔细阅读.
 
shell主要由两部分组成: 
1. 解析shell命令
2. 执行它们 
shell只能够识别简单的shell命令,比如下面这些:
```
[t.sh]
ls > y
cat < y | sort | uniq | wc > y1
cat y1
rm y1
ls | sort | uniq | wc
rm y
```

## 编译 
使用gcc编译sh.c,并尝试执行`./a.out < t.sh`.因为我们还有些Feature没有实现,因此会报错.

# 特性
下面我们来依次实现缺失的特性.

## 执行简单命令
源码中已经提供了解析部分的代码,我们只需要填充runcmd下的代码即可.
这里主要是考察`execv`和`execvp`的用法.

## IO重定向

## 实现管道

## 更多地挑战
1. 实现多个命令串联,通过`;`隔开.
2. 通过实现小括号,实现子shell.
3. 通过支持`&`和`wait`,实现命令后台运行.

所有这些功能都是通过修改`parser`和`runcmd`实现的.