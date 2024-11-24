---
title: nm
tags: [Linux工具]
categories: [Linux工具]
permalink: 
poster:
  topic: null
  headline: null
  caption: null
  color: null
date: 2024-11-24 23:17:52
topic: linux_tool
description:
cover:
banner:
references:
---
nm命令是linux下针对某些特定文件的分析工具，能够列出库文件（.a、.lib）、目标文件（\*.o）、可执行文件的符号表。

## 一、nm命令的常用参数

* -A 或 -o 或 --print-file-name：打印出每个符号属于的文件
* -a 或 --debug-syms：显示调试符号。
* -B：等同于–format=bsd，用来兼容MIPS的nm。
* -C 或 --demangle：将低级符号名解码(demangle)成用户级名字。这样可以使得C++函数名具有可读性。
* -D 或 --dynamic：显示动态符号。该任选项仅对于动态目标(例如特定类型的共享库)有意义。
* -f forma 或 --format=formatt：使用format格式输出。format可以选取bsd、sysv或posix，该选项在GNU的nm中有用。默认为bsd。
* -g 或 --extern-only：仅显示外部符号。
* -n 、-v 或 --numeric-sort：按符号对应地址的顺序排序，而非按符号名的字符顺序。
* -p 或 --no-sort：按目标文件中遇到的符号顺序显示，不排序。
* -P 或 --portability：使用POSIX.2标准输出格式代替默认的输出格式。等同于使用任选项-f posix。
* -s 或 --print-armap：当列出库中成员的符号时，包含索引。索引的内容包含：哪些模块包含哪些名字的映射。
* -r 或 --reverse-sort：反转排序的顺序(例如，升序变为降序)。
* --size-sort：按大小排列符号顺序。该大小是按照一个符号的值与它下一个符号的值进行计算的。
* -t radix 或 --radix=radix：使用radix进制显示符号值。radix只能为“d”表示十进制、“o”表示八进制或“x”表示十六进制。
* --target=bfdname：指定一个目标代码的格式，而非使用系统的默认格式。
* -u 或 --undefined-only：仅显示没有定义的符号(那些外部符号)。
* -l 或 --line-numbers：对每个符号，使用调试信息来试图找到文件名和行号。对于已定义的符号，查找符号地址的行号。对于未定义符号，查找指向符号重定位入口的行号。如果可以找到行号信息，显示在符号信息之后。
* -V 或 --version：显示nm的版本号。
* --help：显示nm的任选项。

举个栗子

编写源文件test.c

```C
#include <stdio.h>

const char ch = 'x';
int uninit;
int init = 10;

void function() {
    int *ref = &init;
    static int sta_int = 10;
    printf("%c", ch);
}
```

编译test.c文件 生成test.o

`gcc -c test.c`

用nm命令分析符号表

`nm -n test.o`（-n以地址排序，方便查看）

输出结果：

```C
                 U putchar
0000000000000000 R ch
0000000000000000 T function
0000000000000000 D init
0000000000000004 d sta_int.2182
0000000000000004 C uninit
```

## 二、nm信息的含义

第一列：符号值，即该符号的起始地址
第二列：符号类型，各字母代表什么类型在下一小节中介绍
第三列：符号名称

## 三、输出符号类型详解

符号类型大写代表全局符号，小写代表本地符号

* A 该符号的值是绝对的，在以后的链接过程中，不允许进行改变。这样的符号值，常常出现在中断向量表中，例如用符号来表示各个中断向量函数在中断向量表中的位置。
* B 该符号的值出现在非初始化数据段(bss)中。例如，在一个文件中定义全局static int test。则该符号test的类型为b，位于bss section中。其值表示该符号在bss段中的偏移。一般而言，bss段分配于RAM中。
* C 该符号为common。common symbol是未初始话数据段。该符号没有包含于一个普通section中。只有在链接过程中才进行分配。符号的值表示该符号需要的字节数。例如在一个c文件中，定义int test，并且该符号在别的地方会被引用，则该符号类型即为C。否则其类型为B。
* D 该符号位于初始话数据段中。一般来说，分配到data section中。例如定义全局int baud_table[5] = {9600, 19200, 38400, 57600, 115200}，则会分配于初始化数据段中。
* G 该符号也位于初始化数据段中。主要用于small object提高访问small data object的一种方式。
* I 该符号是对另一个符号的间接引用。
* N 该符号是一个debugging符号。
* R 该符号位于只读数据区。例如定义全局const int test[] = {123, 123};则test就是一个只读数据区的符号。注意在cygwin下如果使用gcc直接编译成MZ格式时，源文件中的test对应_test，并且其符号类型为D，即初始化数据段中。但是如果使用m6812-elf-gcc这样的交叉编译工具，源文件中的test对应目标文件的test,即没有添加下划线，并且其符号类型为R。一般而言，位于rodata section。值得注意的是，如果在一个函数中定义const char *test = “abc”, const char test_int = 3。使用nm都不会得到符号信息，但是字符串“abc”分配于只读存储器中，test在rodata section中，大小为4。
* S 符号位于非初始化数据区，用于small object。
* T 该符号位于代码区text section。
* U 该符号在当前文件中是未定义的，即该符号的定义在别的文件中。例如，当前文件调用另一个文件中定义的函数，在这个被调用的函数在当前就是未定义的；但是在定义它的文件中类型是T。但是对于全局变量来说，在定义它的文件中，其符号类型为C，在使用它的文件中，其类型为U。
* V 该符号是一个weak object。
* W 该符号是没有被明确标记为weak object的弱符号类型。
* 该符号是a.out格式文件中的stabs symbol。
* ? 该符号类型没有定义。