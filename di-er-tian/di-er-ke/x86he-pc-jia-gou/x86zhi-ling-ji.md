# x86指令集
## Intel汇编语法
1. 指令格式: `op dst, src`
 
## AT&T汇编语法
1. 指令格式: `op src, dst`
2. 后缀,规定了指令操作的位宽:
```
b   字节(8位),对应于Intel的byte ptr
w   字(16位),对应于Intel的word ptr
l   双字(实际是表示long,32位),对应于Intel的dword ptr
q   四字(64位),对应于Intel的qword ptr
```

## 操作对象
操作对象:
1. 寄存器
2. 常量
3. 寄存器指向的内存
4. 常量指向的内存

## 示例

|AT&T syntax | "C"-ish equivalent | |
| --- | --- | --- |
| movl %eax, %edx | edx = eax; | register mode |
| movl $0x123, %edx | edx = 0x123; | immediate |
| movl 0x123, %edx | edx = *(int32_t*)0x123; | direct |
| movl (%ebx), %edx | edx = *(int32_t*)ebx; | indirect |
| movl 4(%ebx), %edx | edx = *(int32_t*)(ebx+4); | displaced |












----