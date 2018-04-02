# PC虚拟机

## 虚拟机
1. 和真实PC的行为一致.
2. 仅仅由软件实现,而非硬件.
3. 从宿主机看来,虚拟机仅仅是一个正常的进程.

## 硬件模拟
使用进程用户空间内存来保存模拟的硬件状态
1. 使用全局变量来模拟CPU寄存器
```
		int32_t regs[8];
		#define REG_EAX 1;
		#define REG_EBX 2;
		#define REG_ECX 3;
		...
		int32_t eip;
		int16_t segregs[4];
		...
```










---