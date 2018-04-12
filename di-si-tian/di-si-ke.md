# 第四课 shell 和 操作系统结构

# 作业
## 回顾作业shell
1. `exec`为什么需要两个参数
2. `exec`进程结束后会发生什么?
3. `execv()`会返回么? 不会
4. shell在执行完一条命令后,怎么继续?
5. `exec`是否如何实现重定向的?
6. 子进程的`fd tables`被修改会影响到主进程么?
7. `ls | wc -l`
   * 如果`ls`的输出超过`wc`的接收,会怎么样?
   * 如果`ls`的输出慢于`wc`的接收,会怎么样?
   * 每条子命令何时退出?
   * 如果`wc`没有关闭写管道会如何?
   * 如果`ls`没有关闭读管道会如何?
8. 内核如何决定何时清理一个pipe buffer?
9. shell怎么知道一个管道命令结束?`ls | sort | tail` wait?
10. 管道命令中可以尝试减少fork的次数么?
   * 尝试执行`pcmd->left`不fork.
   * 尝试执行`runcmd`不fork.
   * 尝试执行`pcmd->right`不fork.这时执行`sleep 10 | echo hi`,看看有什么变化.
11. 管道命令中,为什么要在两个子进程都开始后再调用两次wait(). 如果将wait()移到两次fork之间会发什么?试试`ls | wc -l cat < big | wc -l`
12. 核心: 系统调用短小精悍,通过组合使用这些系统调用,从而达到各种各样的目的.

##　附加题shell
1. 如何实现顺序执行`;`.为什么必须要在`scmd->right`前wait()?
2. 如何实现后台执行`&`?当后台执行命令退出时,恰好sh正在等待一个前台进程退出,会怎么样?
3. 如何实现嵌套?`(echo a; echo b) | wc -l`.
4. 下面两个命令有什么区别?什么机制避免了内容覆盖?
   * echo a > x ; echo b > x
   * (echo a; echo b) > x 共享fd偏移

## Unix系统调用

### fork和exec
`fork`和`exec`分开,咋一看比较多余.`fork`拷贝了内存,`exec`又丢弃了这些内存,重新载入.为什么不用一个`api`搞定呢?像这样`pid = forkexec(path, argv, fd0, df1)`

事实上,分离了`fork`和`exec`是有意义的.这样在`fork`和`exec`之间,我们可以插入自己的代码.比如重定向的功能就依赖于此.`fork(); IO redirection; exec()`

另外,`fork()`的消耗很小.

### 文件描述符
文件描述符是一种间接抽象.进程的实际IO细节被隐藏在内核之中.在`fork`和`exec`时,会被保留.

文件描述符有助于程序抽象,不再需要关心fd代表的是文件,控制台或者是管道.

### 管道的必要性
为什么要设计管道,而不是使用临时文件替代?
` ls > tempfile ; wc -l < tempfile`
管道自管理自回收.

### 系统调用设计























---