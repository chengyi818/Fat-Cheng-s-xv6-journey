# Part2 虚拟内存

## Exercise2 
熟悉x86的保护模式架构,主要分为分段和分页两种机制.

阅读[Intel 80386 Reference Programmer's Manual](https://pdos.csail.mit.edu/6.828/2017/readings/i386/toc.htm)中的第5和第6章.尤其是*5.2 Page Translation*和*6.4 Page-Level Protection*两小节.

## 虚拟地址, 线性地址, 物理地址
x86中的*虚拟地址*由段地址和段内偏移两部分组成.

*线性地址*是进行了段转换而没有进行页转换前的地址.

*物理地址*是最终进行了段转换和页转换后的地址,也是最终在硬件总线上对内存寻址的地址.

示意图如下:

```

           Selector  +--------------+         +-----------+
          ---------->|              |         |           |
                     | Segmentation |         |  Paging   |
Software             |              |-------->|           |---------->  RAM
            Offset   |  Mechanism   |         | Mechanism |
          ---------->|              |         |           |
                     +--------------+         +-----------+
            Virtual                   Linear                Physical
```

