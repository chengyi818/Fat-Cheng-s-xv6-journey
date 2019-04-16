# homework mmap()

# 背景
本练习的主要目的在于通过Unix系统接口进一步熟悉如何管理用户程序中的虚拟内存.本练习需要在支持Unix API的PC上完成,Linux,Mac均可.

首先,下载[mmap.c](https://pdos.csail.mit.edu/6.828/2017/homework/mmap.c)

这段代码在虚拟内存中保存了一份非常大的平方根值表.这张表太大了,以至于物理内存无法容纳.正确地作法是,根据发生`page fault`的页面地址动态地计算平方根值.我们的任务是就是实现这样的动态计算,我们需要用到`signal handle`和Unix内存映射系统调用.鉴于物理内存的限制,建议当发生`page fault`时,可以使用取消最后一页映射的简单策略.

# 编译和执行
编译`mmap.c`,命令如下:
```
$ gcc mmap.c -lm -o mmap
```

执行结果如下:
```
$ ./mmap
page_size is 4096
Validating square root table contents...
oops got SIGSEGV at 0x7f6bf7fd7f18
```

---

# 说明
当处理器访问平方根值表时,内存映射并不存在,因此内核会调用到我们注册的信号处理函数`handle_sigsegv()`.
修改该函数:
1. 映射一个内存页到发生`page fault`的地址.
2. 取消前一个内存页的映射,以免突破物理内存限制.
3. 再将相应的平方根值填入新的内存页.
4. 使用函数`calculate_sqrts() `来计算平方根值.

# 测试
程序中已经包含了测试代码,如果一切正常,会打印`All tests passed`.

---

# Tips
可以通过
```
$man mmap
$man munmap
```
来获取相关帮助信息.
