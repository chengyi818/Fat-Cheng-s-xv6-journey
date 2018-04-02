# x86和Pc架构

## PC硬件和x86

这部分请阅读教材提供的PPT资料

## Notes: x86和Pc架构

### 大纲

1. PC架构
2. x86指令集
3. gcc调用规定
4. PC虚拟机

### PC架构

#### PC组成

1. x86 CPU,其中包含一组寄存器,执行单元,内存管理硬件.
2. CPU芯片管脚,其中包含地址和数据信号.
3. 内存
4. 硬盘
5. 键盘
6. 显示器
7. 其他资源: 只读BIOS, 时钟等.

#### 启动

我们将从1978年推出的16位的8086 CPU开始.

#### CPU指令执行

只要不出现改变EIP寄存器的指令,EIP将永远递增,不断执行下一条指令.

#### 工作空间: 数据寄存器

1. 4个16位数据寄存器: AX, BX, CX, DX
2. 每个都可以分为高低两个,比如: AH和AL.
3. 寄存器速度非常快,也非常少.

#### 更大的工作空间: 内存

1. CPU通过地址线来控制访问地址.
2. 数据则由数据线传递.

#### 地址寄存器: 指向内存
1. SP: 栈顶寄存器
2. BP: 栈底寄存器
3. SI: Source index,源变址寄存器
4. DI: Destination index,目的变址寄存器

#### 指令位置
1. 指令保存在内存中
2. IP: instruction pointer, 程序计数器
3. 通常自增
4. 可以被如下指令修改: CALL, RET, JMP, 条件跳转.

#### 条件跳转
##### FLAGS标志位
1. 算术操作溢出
2. 正负
3. 零/非零
4. 加减的借位
5. 中断
6. 直接数据拷贝

##### 相关指令
1. JP, JN, J[N]Z, J[N]C, J[N]O.

#### I/O
##### 原始PC架构: 专用IO空间
1. 和内存操作相似,但使用IO信号.
2. 只有1024个IO地址
3. 使用特定指令操作(IN, OUT)
4. 示例,写一个字节到line printer:
```
#define DATA_PORT    0x378
#define STATUS_PORT  0x379
#define   BUSY 0x80
#define CONTROL_PORT 0x37A
#define   STROBE 0x01
void
lpt_putc(int c)
{
  /* wait for printer to consume previous byte */
  while((inb(STATUS_PORT) & BUSY) == 0)
    ;

  /* put the byte on the parallel lines */
  outb(DATA_PORT, c);

  /* tell the printer to look at the data */
  outb(CONTROL_PORT, STROBE);
  outb(CONTROL_PORT, 0);
}
```

##### 现代PC架构: 内存映射IO
1. 使用正常的物理内存地址

  1.1 突破了IO地址空间的限制
  1.2 不需要特殊的操作指令
  1.3 系统控制器将在对应的设备间充当路由功能.
  
2. 操作设备如同操作一块神奇的内存

  2.1 寻址,访问等操作和操作内存一样
  2.2 操作结果和内存不同
  2.3 读写设备映射内存均会导致不同的反应
  2.4 外部事件变化可能会导致读取的结果发生变化
  
#### 内存限制突破















---



