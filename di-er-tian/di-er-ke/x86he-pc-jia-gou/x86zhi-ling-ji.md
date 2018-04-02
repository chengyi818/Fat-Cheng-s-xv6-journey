# x86指令集
## Intel汇编语法
1. 指令格式: `op dst, src`
 
## AT&T汇编语法
1. 指令格式: `op src, dst`
2. 后缀:
```
b   字节(8位),对应于Intel的byte ptr
w   字(16位),对应于Intel的word ptr
l   双字(实际是表示long,32位),对应于Intel的dword ptr
q   四字(64位),对应于Intel的qword ptr
```