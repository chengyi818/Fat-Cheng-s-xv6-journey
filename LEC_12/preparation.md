# 课前准备
1. 阅读文档第六章: 文件系统.log相关的内容可以跳过.
2. 阅读xv6源码中, `bio.c`, `fs.c`, `sysfile.c`以及`file.c`.

---

```
+-------------------+
|  File Descriptor  |
+-------------------+
|  Pathname         |
+-------------------+
|  Directory        |
+-------------------+
|  Inode            |
+-------------------+
|  Logging          |
+-------------------+
|  Buffer cache     |
+-------------------+
|  Disk             |
+-------------------+
```

![xv6磁盘存储结构](../images/xv6_disk_structure.png)

---

# 文件系统

## 目的
1. 组织和存储数据
2. 在用户和进程间共享数据.
3. 持久化存储

## 挑战
1. 在磁盘上,持久存储文件树结构
2. 记录每个文件所使用的磁盘block
3. 记录磁盘上空闲的区域
4. 文件系统必须支持crash recovery.
5. 不同进程同时操作文件系统的并发问题
6. IDE读取速度远慢于内存,因此必须缓存常用的block.

---



---

# 关键词
1. 持久化 persistence
2. 崩溃恢复 crash recovery
3. 事务处理 transaction
4. vfs模型 inode

---

1.  bio.c
buffer cache layer

2. fs.c
inode layer
directory layer

3. sysfile.c
用户进程操作文件系统 调用接口

4. file.c
file descriptor layer

---
