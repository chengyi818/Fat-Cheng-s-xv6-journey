# preparation
1. 阅读journal_design.pdf
2. 第6页左栏中间,提到: 在没有将buffers写回磁盘前,我们不能删除日志系统中这些buffer的备份.请举例说明如果删除了,会导致怎么样的后果?

---
# 小评
扩展了ext2fs的功能,基本和xv6的logging设计类似.
