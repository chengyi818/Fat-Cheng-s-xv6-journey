# gcc x86调用约定

## 栈增长方向
在x86中,栈都是向下增长的.

| Example instruction | What it does |
| --- | --- |
| pushl %eax | 	subl $4, %esp <br> movl %eax, (%esp) | 
| popl %eax | movl (%esp), %eax <br> addl $4, %esp | 
| call 0x12345 | pushl %eip(\*) <br> movl $0x12345, %eip (\*) | 
| ret | popl %eip (\*) |

其中带*号的指令并非实际存在的指令,只是演示作用.

## GCC调用约定
GCC编译器规定了栈的使用方式,在x86上调用者和被调用者如何使用内存栈.

### 函数入口处
在执行完call指令后:
1. eip: 指向函数的第一条指令
2. esp+4: 指向第一个参数
3. esp: 指向返回地址

### 函数返回后
在执行完ret指令后:
1. eip: 指向返回地址
2. esp: 指向调用者入参后的地址
3. 栈上可能保留着被调用函数不用的参数
4. eax: 保存着返回值.如果是64位的返回值,则edx也会被使用到.
5. eax,edx,ecx可能会丢弃?(FC批注: 不是很理解)
6. 执行指令call时,%ebp,%ebx,%esi,%edi必须有值.

### 术语
1. %eax, %ecx, %edx 是调用者保存寄存器
2. %ebp, %ebx, %esi, %edi 是被调动者保存寄存器
FC批注: 这个术语不是很理解.


## 函数栈结构
只要不破坏Gcc的约定,函数可以做任何事情.

```
		       +------------+   |
		       | arg 2      |   \
		       +------------+    >- previous function's stack frame
		       | arg 1      |   /
		       +------------+   |
		       | ret %eip   |   /
		       +============+   
		       | saved %ebp |   \
		%ebp-> +------------+   |
		       |            |   |
		       |   local    |   \
		       | variables, |    >- current function's stack frame
		       |    etc.    |   /
		       |            |   |
		       |            |   |
		%esp-> +------------+   /
		
```
1. %esp可以向下增长,使得当前函数的可用栈变大.
2. %ebp指向保存上一个函数%ebp的内存地址.通过%ebp,我们可以遍历整个函数调用栈.
3. 函数入口通常指令为:
```
			pushl %ebp
			movl %esp, %ebp
```
或者`enter $0, $0`.后面一种形式很少使用.
4. 函数出口指令通常为:
```
			movl %ebp, %esp
			popl %ebp
``` 
或者`leave`.后面一种指令更经常被使用.

## 示例
### C代码如下:
```
		int main(void) { return f(8)+1; }
		int f(int x) { return g(x); }
		int g(int x) { return x+3; }
```
### 汇编指令如下:
```
		_main:
					prologue
			pushl %ebp
			movl %esp, %ebp
					body
			pushl $8
			call _f
			addl $1, %eax
					epilogue
			movl %ebp, %esp
			popl %ebp
			ret
		_f:
					prologue
			pushl %ebp
			movl %esp, %ebp
					body
			pushl 8(%esp)
			call _g
					epilogue
			movl %ebp, %esp
			popl %ebp
			ret

		_g:
					prologue
			pushl %ebp
			movl %esp, %ebp
					save %ebx
			pushl %ebx
					body
			movl 8(%ebp), %ebx
			addl $3, %ebx
			movl %ebx, %eax
					restore %ebx
			popl %ebx
					epilogue
			movl %ebp, %esp
			popl %ebp
			ret
```

### 精简_g指令:
```
		_g:
			movl 4(%esp), %eax
			addl $3, %eax
			ret
```

### 精简_f指令:
因为\_f没有任何实际意义,所以可以直接跳转到\_g即可.

### 编译,链接,装载
1. 预处理:
以ASCII码文本为输入,展开头文件,宏定义等信息.输出C源码.

2. 编译器:
以预处理输出为输入,对C源码进行编译,输出ASCII码的汇编指令.

3. 汇编器:
以编译器输出为输入,将汇编指令转换为机器指令,输出二进制object文件.

4. 链接器:
以多个二进制object文件为输入,输出一个二进制程序image.

5. 装载器:
运行时,将二进制程序image载入内存中,并跳转到程序入口执行.












----