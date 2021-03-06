# 屏障barrier

本次作业中我们将通过`pthread`库提过了条件变量来实现屏障功能.简单来说,屏障是程序运行中的一个点,必须等待所有线程均达到此点后,所有线程才能继续向下执行.详细解释可以参考[wiki barrier](https://en.wikipedia.org/wiki/Barrier_(computer_science))

条件变量`Condition variables`是一种同步技术,类似于xv6中的`sleep`和`wakeup`.

## 源码和编译

下载[barrier.c](https://pdos.csail.mit.edu/6.828/2017/homework/barrier.c),并按如下方式编译:
```
$ gcc -g -O2 -pthread barrier.c
$ ./a.out 2
Assertion failed: (i == t), function thread, file barrier.c, line 55.
```

参数2表示屏障上的线程数.每个线程都处于循环之中,不断地调用`barrier()`,然后睡眠几十微秒.
只有在一个线程在另一个线程到达屏障前,离开屏障,`assert`断言才会触发.而我们的目标是所有线程到达屏障后均阻塞,直到最后一个线程到达屏障.

## 目标

我们需要为`barrier()`函数添加代码.除了用到`pthread_mutex_lock()`之外,我们还会用到和条件变量相关的两个函数:
1. pthread_cond_wait(&cond, &mutex);  // go to sleep on cond, releasing lock mutex
2. pthread_cond_broadcast(&cond);     // wake up every thread sleeping on cond

其中,`pthread_cond_wait()`被调用后,会释放`mutex lock`.在返回时,会再次获取该锁.可以参考xv6`sleep`的实现.

## 难点
主要难点如下:
1. 我们必须处理屏障barrier调用,每一轮`bstate.round`必须加一.
2. 同时,我们必须保证所有线程不能在还有线程未到达屏障时离开屏障.因为我们会不断复用`bstate.nthread`.务必注意此处的竞争问题.



## 测试用例
```
$./a.out 1
OK; passed
$./a.out 2
OK; passed
$./a.out 4
OK; passed
```