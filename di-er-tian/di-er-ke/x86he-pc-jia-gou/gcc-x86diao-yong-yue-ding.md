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

## 具体使用
只要不破坏Gcc的约定,函数可以做任何事情.

### 函数栈结构
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
















----