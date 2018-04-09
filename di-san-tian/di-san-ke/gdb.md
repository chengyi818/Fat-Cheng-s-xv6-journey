# gdb使用
资料位置`resources/gdb_slides.pdf`

# Tips
1. 帮助,`help <command-name>`
2. 缩写: `c = co = cont = continue`, `s = step`, `si = stepi`
3. 回车键将会重复上次命令

# 常用命令
1. step
    单步执行代码,遇到函数将会进入.
2. next
    单步执行代码,遇到函数不会进入.
3. stepi
    单步执行指令,遇到函数将会进入.
4. nexti
    单步执行指令,遇到函数不会进入.
5. continue
    执行代码直到遇到断点,或者按下Ctrl-c.
6. finish
    执行代码,直到当前函数执行完毕.
7. advance *location*
    运行代码到指定位置
8. break *location*
    在指定位置设下断点
9. 位置可以是以下形式:
    "*0x7c00", "mon_backtrace", "monitor.c:71"
10. 可以通过以下命令修改断点: delete, disable, enable
11. watch *expression*
    观测点.当表达式的值变化时,停止运行.
12. watch -l *address*
    观测点,当地址中的值变化时,停止运行.
13. x/x *address*
    以16进制形式,打印内存值
14. x/i *address*
    以指令形式,打印内存值
15. print
    执行C表达式,并按合适的格式打印
16. `p *((struct elfhdr *) 0x10000)`显然要比`x/13x 0x10000`直观的多
17. info registers
    显示所有寄存器的值
18. info frame
    打印当前函数栈
19. list *location*
    显示执行位置的源码
20. backtrace
    回溯函数栈
21. set *variable*
    执行期间改变变量的值
22. 
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
---
