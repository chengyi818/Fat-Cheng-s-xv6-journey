# PC虚拟机

## 虚拟机
1. 和真实PC的行为一致.
2. 仅仅由软件实现,而非硬件.
3. 从宿主机看来,虚拟机仅仅是一个正常的进程.

## 硬件模拟
使用进程用户空间内存来保存模拟的硬件状态
* 使用全局变量来模拟CPU寄存器
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

* 使用用户空间内存来模拟物理内存
```
char mem[256*1024*1024];
```

## 指令执行模拟
我们通过不断读取指令,执行指令,从而在用户空间模拟CPU执行指令.
```
	for (;;) {
		read_instruction();
		switch (decode_instruction_opcode()) {
		case OPCODE_ADD:
			int src = decode_src_reg();
			int dst = decode_dst_reg();
			regs[dst] = regs[dst] + regs[src];
			break;
		case OPCODE_SUB:
			int src = decode_src_reg();
			int dst = decode_dst_reg();
			regs[dst] = regs[dst] - regs[src];
			break;
		...
		}
		eip += instruction_length;
	}
```

## 内存映射模拟
通过解析虚拟机中的"物理"内存地址,来模拟内存映射
```
	#define KB		1024
	#define MB		1024*1024

	#define LOW_MEMORY	640*KB
	#define EXT_MEMORY	10*MB

	uint8_t low_mem[LOW_MEMORY];
	uint8_t ext_mem[EXT_MEMORY];
	uint8_t bios_rom[64*KB];

	uint8_t read_byte(uint32_t phys_addr) {
		if (phys_addr < LOW_MEMORY)
			return low_mem[phys_addr];
		else if (phys_addr >= 960*KB && phys_addr < 1*MB)
			return rom_bios[phys_addr - 960*KB];
		else if (phys_addr >= 1*MB && phys_addr < 1*MB+EXT_MEMORY) {
			return ext_mem[phys_addr-1*MB];
		else ...
	}

	void write_byte(uint32_t phys_addr, uint8_t val) {
		if (phys_addr < LOW_MEMORY)
			low_mem[phys_addr] = val;
		else if (phys_addr >= 960*KB && phys_addr < 1*MB)
			; /* ignore attempted write to ROM! */
		else if (phys_addr >= 1*MB && phys_addr < 1*MB+EXT_MEMORY) {
			ext_mem[phys_addr-1*MB] = val;
		else ...
	}
```

## IO设备模拟
通过侦测对特殊内存空间的访问,来模拟正确的IO行为.
1. 读写虚拟硬盘,将被模拟为: 读写宿主机上的一个文件.
2. 输出到VGA显示器,将被模拟为: 在X windows中输出
3. 键盘输入,将被模拟为: 从X输入事情队列中读取





---