---
title: readelf
tags: [Linux工具]
categories: [Linux工具]
permalink: 
poster:
  topic: null
  headline: null
  caption: null
  color: null
date: 2024-11-24 23:16:27
topic: linux_tool
description:
cover:
banner:
references:
---
## 一、三类目标文件（ELF）：

1.可重定位目标文件 (.o )

每个 .o 文件都是由对应的 .c 文件通过编译器和汇编器生成；包含代码和数据，代码和数据地址都从0开始。通过 gcc -c xxx.c 得到。

2.可执行目标文件（默认为a.out）

由链接器生成，包含的代码和数据可以直接通过加载器加载到内存中并被执行。通过gcc -o xxx.c 得到。

3.共享目标文件 (.so）

特殊的可重定位目标文件，可以在链接(静态共享库)时加入目标文件或加载时或运行时(动态共享库)被动态的加载到内存并执行。在 windows 中被称为 Dynamic Link Libraries(DLLs)。 gcc xxx.c -fPIC -shared -o libxxx.so （-fPIC 作用于编译阶段，告诉编译器产生与位置无关代码。）

readelf命令：

通过readelf来区分上面三种类型的ELF文件，每种类型文件的头部信息是不一样的。

## 二、readelf -h

readelf -h main.o -h等价于--file-header

```Bash
  1 ELF Header:
  2   Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  3   Class:                             ELF64
  4   Data:                              2's complement, little endian
  5   Version:                           1 (current)
  6   OS/ABI:                            UNIX - System V
  7   ABI Version:                       0
  8   Type:                              REL (Relocatable file)
  9   Machine:                           Advanced Micro Devices X86-64
 10   Version:                           0x1
 11   Entry point address:               0x0
 12   Start of program headers:          0 (bytes into file)
 13   Start of section headers:          720 (bytes into file)
 14   Flags:                             0x0
 15   Size of this header:               64 (bytes)
 16   Size of program headers:           0 (bytes)
 17   Number of program headers:         0
 18   Size of section headers:           64 (bytes)
 19   Number of section headers:         12
 20   Section header string table index: 11
```

* 第 1 行，ELF Header: 指名 ELF 文件头开始。
* 第 2 行，Magic 魔数，用来指名该文件是一个 ELF 目标文件。第一个字节 7F 是个固定的数；后面的 3 个字节正是 E, L, F 三个字母的 ASCII 形式。
* 第 3 行，CLASS 表示文件类型，这里是 64位的 ELF 格式。
* 第 4 行，Data 表示文件中的数据是按照什么格式组织(大端或小端)的，不同处理器平台数据组织格式可能就不同，如x86平台为小端存储格式。
* 第 5 行，当前 ELF 文件头版本号，这里版本号为 1 。
* 第 6 行，OS/ABI ，指出操作系统类型，ABI 是 Application Binary Interface 的缩写。
* 第 7 行，ABI 版本号，当前为 0 。
* 第 8 行，Type 表示文件类型。ELF 文件有 3 种类型，一种是如上所示的 Relocatable file 可重定位目标文件，一种是可执行文件(Executable)，另外一种是共享库(Shared Library) 。 [这里就是区分上面三种类型的ELF文件]
* 第 9 行，机器平台类型，这里是在X86-64位机器。
* 第 10 行，当前目标文件的版本号。
* 第 11 行，程序的虚拟地址入口点，因为这还不是可运行的程序，故而这里为零。如果是可运行程序，这个地址并不是main函数的地址，而是_start函数的地址，_start由链接器创建，_start是为了初始化程序。通过这个命令可以看到_start函数，objdump -d -j .text a.out(默认，改名之后需更改此处)。
* 第 12 行，与 11 行同理，这个目标文件没有 Program Headers。
* 第 13 行，sections 头开始处，这里 720 是十进制，表示从地址偏移 0x450 处开始。
* 第 14 行，是一个与处理器相关联的标志，x86 平台上该处为 0 。
* 第 15 行，ELF 文件头的字节数。64bytes
* 第 16 行，因为这个不是可执行程序，故此处大小为 0。
* 第 17 行，同理于第 16 行。
* 第 18 行，sections header 的大小，这里每个 section 头大小为 64bytes。
* 第 19 行，一共有多少个 section 头，这里是 12个。
* 第 20 行，section 头字符串表索引号。区中存储的信息是用来链接使用的，主要包括：程序代码、程序数据（变量）、重定向信息等。比如：Code section保存的是代码，data section保存的是初始化或未初始化的数据，等等。

## 三、readelf -S main

功能：查看区内容

```Bash
 1 There are 29 section headers, starting at offset 0x1948:
  2 Section Headers:
  3   [Nr] Name              Type             Address           Offset      Size              EntSize          Flags  Link  Info  Align
  4   [ 0]                   NULL             0000000000000000  00000000       0000000000000000  0000000000000000           0     0     0
  5   [ 1] .interp           PROGBITS         0000000000000238  00000238       000000000000001c  0000000000000000   A       0     0     1
  6   [ 2] .note.ABI-tag     NOTE             0000000000000254  00000254       0000000000000020  0000000000000000   A       0     0     4
  7   [ 3] .note.gnu.build-i NOTE             0000000000000274  00000274     0000000000000024  0000000000000000   A       0     0     4
  8   [ 4] .gnu.hash         GNU_HASH         0000000000000298  00000298       000000000000001c  0000000000000000   A       5     0     8
  9   [ 5] .dynsym           DYNSYM           00000000000002b8  000002b8       0000000000000090  0000000000000018   A       6     1     8
 10   [ 6] .dynstr           STRTAB           0000000000000348  00000348       000000000000007d  0000000000000000   A       0     0     1
 11   [ 7] .gnu.version      VERSYM           00000000000003c6  000003c6     000000000000000c  0000000000000002   A       5     0     2
 12   [ 8] .gnu.version_r    VERNEED          00000000000003d8  000003d8       0000000000000020  0000000000000000   A       6     1     8
 13   [ 9] .rela.dyn         RELA             00000000000003f8  000003f8       00000000000000c0  0000000000000018   A       5     0     8
 14   [10] .init             PROGBITS         00000000000004b8  000004b8     0000000000000017  0000000000000000  AX       0     0     4
 15   [11] .plt              PROGBITS         00000000000004d0  000004d0      0000000000000010  0000000000000010  AX       0     0     16
 16   [12] .plt.got          PROGBITS         00000000000004e0  000004e0       0000000000000008  0000000000000008  AX       0     0     8
 17   [13] .text             PROGBITS         00000000000004f0  000004f0       00000000000001e2  0000000000000000  AX       0     0     16
 18   [14] .fini             PROGBITS         00000000000006d4  000006d4     0000000000000009  0000000000000000  AX       0     0     4
 19   [15] .rodata           PROGBITS         00000000000006e0  000006e0       0000000000000004  0000000000000004  AM       0     0     4
 20   [16] .eh_frame_hdr     PROGBITS         00000000000006e4  000006e4      0000000000000044  0000000000000000   A       0     0     4
 21   [17] .eh_frame         PROGBITS         0000000000000728  00000728      0000000000000128  0000000000000000   A       0     0     8
 22   [18] .init_array       INIT_ARRAY       0000000000200df0  00000df0      0000000000000008  0000000000000008  WA       0     0     8
 23   [19] .fini_array       FINI_ARRAY       0000000000200df8  00000df8     0000000000000008  0000000000000008  WA       0     0     8
 24   [20] .dynamic          DYNAMIC          0000000000200e00  00000e00      00000000000001c0  0000000000000010  WA       6     0     8
 25   [21] .got              PROGBITS         0000000000200fc0  00000fc0      0000000000000040  0000000000000008  WA       0     0     8
 26   [22] .data             PROGBITS         0000000000201000  00001000      0000000000000018  0000000000000000  WA       0     0     8
 27   [23] .bss              NOBITS           0000000000201018  00001018       0000000000000008  0000000000000000  WA       0     0     1
 28   [24] .comment          PROGBITS         0000000000000000  00001018       000000000000002b  0000000000000001  MS       0     0     1
 29   [25] .symtab           SYMTAB           0000000000000000  00001048      0000000000000600  0000000000000018          26    43     8
 30   [26] .strtab           STRTAB           0000000000000000  00001648       0000000000000200  0000000000000000           0     0     1
 31   [27] .shstrtab         STRTAB           0000000000000000  00001848      00000000000000f9  0000000000000000           0     0     1
 32 Key to Flags:
 33   W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
 34   L (link order), O (extra OS processing required), G (group), T (TLS),
 35   C (compressed), x (unknown), o (OS specific), E (exclude),
 36   l (large), p (processor specific)
```

* .text：已编译程序的机器代码（二进制指令），该区的标志为X表示可执行。
* .rodata：只读数据，比如printf语句中的格式串和开关（switch）语句的跳转表。
* .data：已初始化的全局C变量。局部C变量在运行时被保存在栈中，既不出现在.data中，也不出现在.bss节中。
* .bss：未初始化的全局C变量。在目标文件中并没有分配实际的空间给它，它只是一个占位符。目标文件格式区分初始化和未初始化变量是为了空间效率在：在目标文件中，未初始化变量不需要占据任何实际的磁盘空间。
* .symtab：一个符号表（symbol table），它存放在程序中被定义和引用的函数和全局变量的信息。一些程序员错误地认为必须通过-g选项来编译一个程序，得到符号表信息。实际上，每个可重定位目标文件在.symtab中都有一张符号表。然而，和编译器中的符号表不同，.symtab符号表不包含局部变量的表目。
* .rel.text：当链接噐把这个目标文件和其他文件结合时，.text节中的许多位置都需要修改。一般而言，任何调用外部函数或者引用全局变量的指令都需要修改。另一方面调用本地函数的指令则不需要修改。注意，可执行目标文件中并不需要重定位信息，因此通常省略，除非使用者显式地指示链接器包含这些信息。
* .rel.data：被模块定义或引用的任何全局变量的信息。一般而言，任何已初始化全局变量的初始值是全局变量或者外部定义函数的地址都需要被修改。
* .strtab：一个字符串表，其内容包括.symtab和.debug节中的符号表，以及节头部中的节名字。字符串表就是以null结尾的字符串序列。
* .init和.fini保存了进程初始化和结束所用的代码，这通常是由编译器自动添加的。

## 四、readelf -s

readelf -s main.o

功能：查看符号表，Value的值是符号的地址。

```C
Symbol table '.symtab' contains 12 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS main.c
     2: 0000000000000000     0 SECTION LOCAL  DEFAULT    1
     3: 0000000000000000     0 SECTION LOCAL  DEFAULT    3
     4: 0000000000000000     0 SECTION LOCAL  DEFAULT    4
     5: 0000000000000000     0 SECTION LOCAL  DEFAULT    6
     6: 0000000000000000     0 SECTION LOCAL  DEFAULT    7
     7: 0000000000000000     0 SECTION LOCAL  DEFAULT    5
     8: 0000000000000000     8 OBJECT  GLOBAL DEFAULT    3 array
     9: 0000000000000000    33 FUNC    GLOBAL DEFAULT    1 main
    10: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND _GLOBAL_OFFSET_TABLE_
    11: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND sum
```

符号表保存了程序实现或使用的所有全局变量和函数，如果程序引用一个自身代码未定义的符号，则称之为未定义符号，这类引用必须在静态链接期间用其他目标模块或库解决，或在加载时通过动态链接解决。