# JOS虚拟内存布局
```
    4 Gig -------->  +------------------------------+
                     |                              | RW/--
                     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
                     :              .               :
                     :              .               :
                     :              .               :
                     |~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~| RW/--
                     |                              | RW/--
                     |   Remapped Physical Memory   | RW/--
                     |                              | RW/--
    KERNBASE, ---->  +------------------------------+ 0xf0000000      --+
    KSTACKTOP        |     CPU0's Kernel Stack      | RW/--  KSTKSIZE   |
                     | - - - - - - - - - - - - - - -|                   |
                     |      Invalid Memory (*)      | --/--  KSTKGAP    |
                     +------------------------------+                   |
                     |     CPU1's Kernel Stack      | RW/--  KSTKSIZE   |
                     | - - - - - - - - - - - - - - -|                 PTSIZE
                     |      Invalid Memory (*)      | --/--  KSTKGAP    |
                     +------------------------------+                   |
                     :              .               :                   |
                     :              .               :                   |
    MMIOLIM ------>  +------------------------------+ 0xefc00000      --+
                     |       Memory-mapped I/O      | RW/--  PTSIZE
 ULIM, MMIOBASE -->  +------------------------------+ 0xef800000
                     |  Cur. Page Table (User R-)   | R-/R-  PTSIZE
    UVPT      ---->  +------------------------------+ 0xef400000
                     |          RO PAGES            | R-/R-  PTSIZE
    UPAGES    ---->  +------------------------------+ 0xef000000
                     |           RO ENVS            | R-/R-  PTSIZE
 UTOP,UENVS ------>  +------------------------------+ 0xeec00000
 UXSTACKTOP -/       |     User Exception Stack     | RW/RW  PGSIZE
                     +------------------------------+ 0xeebff000
                     |       Empty Memory (*)       | --/--  PGSIZE
    USTACKTOP  --->  +------------------------------+ 0xeebfe000
                     |      Normal User Stack       | RW/RW  PGSIZE
                     +------------------------------+ 0xeebfd000
                     |                              |
                     |                              |
                     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
                     .                              .
                     .                              .
                     .                              .
                     |~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~|
                     |     Program Data & Heap      |
    UTEXT -------->  +------------------------------+ 0x00800000
    PFTEMP ------->  |       Empty Memory (*)       |        PTSIZE
                     |                              |
    UTEMP -------->  +------------------------------+ 0x00400000      --+
                     |       Empty Memory (*)       |                   |
                     | - - - - - - - - - - - - - - -|                   |
                     |  User STAB Data (optional)   |                 PTSIZE
    USTABDATA ---->  +------------------------------+ 0x00200000        |
                     |       Empty Memory (*)       |                   |
    0 ------------>  +------------------------------+                 --+
```