# 格式化输出到命令行
大多数人都认为C语言本来就支持`printf`函数,但其实从操作系统的层面,我们必须自己实现所有的IO操作.

仔细阅读`kern/printf.c`,`lib/printfmt.c`,`kern/console.c`,搞清楚它们之间的关系.在后面的课程中,我们会明白为什么`printfmt.c`会在`lib`目录中.

## Exercise 8
我们在实现格式化输出时,有意遗漏了`%o`,也就是8进制输出的实现,请实现.

## 问题
1. 解释`printf.c`和`console.c`之间的接口,特别是`console.c`导出了什么函数,`printf.c`又是如何调用的?

2. 解释`console.c`中的如下代码:
```
1      if (crt_pos >= CRT_SIZE) {
2              int i;
3              memmove(crt_buf, crt_buf + CRT_COLS, (CRT_SIZE - CRT_COLS) * sizeof(uint16_t));
4              for (i = CRT_SIZE - CRT_COLS; i < CRT_SIZE; i++)
5                      crt_buf[i] = 0x0700 | ' ';
6              crt_pos -= CRT_COLS;
7      }
```

3. 单步跟踪执行如下代码,并解释:
```
int x = 1, y = 3, z = 4;
cprintf("x %d, y %x, z %d\n", x, y, z);
```
> 在`cprintf()`中fmt和ap指向哪里?
> 详细说明`cons_putc`,`va_arg`,`vcprintf`三个函数的参数情况.

4.  执行如下代码,并解释输出.如果x86不是小端而是大端,如何修改从而保证输出不变?
```
unsigned int i = 0x00646c72;
cprintf("H%x Wo%s", 57616, &i);
```

5. 下面代码中,`y=`后输出的是什么?为什么?
```
cprintf("x=%d y=%d", 3);
```

6. 如果GCC改变入参规则,根据声明顺序来入参.如何调整`cprintf`,从而确保依然可以输出可变数目的变量?

## 挑战
增加console的特性,从而可以输出彩色的字体.可以在6.828的参考资料中找到一些帮助.
