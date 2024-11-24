---
title: objdump
tags: [Linux工具]
categories: [Linux工具]
permalink: 
poster:
  topic: null
  headline: null
  caption: null
  color: null
date: 2024-11-24 23:19:04
topic: linux_tool
description:
cover:
banner:
references:
---
objdump提供了对二进制文件进行反汇编和查看目标文件信息的能力。用于分析目标文件（object file）和可执行文件（executable file）。它可以显示二进制文件的汇编代码、符号表、段信息等，是理解程序底层实现、调试和逆向工程的有力助手。

## 一、objdump的基本用法

显示目标文件的反汇编代码：

```C
objdump -d your_binary
```

该命令会显示目标文件中所有段的反汇编代码。这是一种深入了解程序执行逻辑的方式。

显示符号表信息：

```C
objdump -t your_binary
```

该命令会显示目标文件的符号表，包括函数名、变量名等信息。

显示文件头信息：

```C
objdump -f your_binary
```

该命令显示目标文件的文件头信息，包括文件格式、入口点地址等。objdump 的使用还可以根据需求加入一些参数来获取更详细的信息。

显示所有段的详细信息：

```C
objdump -p your_binary
```

这将显示目标文件中所有段的详细信息，包括每个段的大小、偏移量等。

显示特定段的反汇编代码：

```C
objdump -s -j section_name your_binary
```

这将显示指定段（section_name）的反汇编代码。

只显示符号表的信息：

```C
objdump -T your_binary
```

该命令显示符号表的信息，但不显示反汇编代码。

显示源代码和反汇编代码：

```C
objdump -S your_binary
```

这将显示源代码和反汇编代码的混合视图，方便理解源代码和汇编之间的对应关系。

以指定格式显示反汇编代码：

```C
objdump -M intel -d your_binary
```

-M 参数允许你指定反汇编代码的输出格式，例如 intel 或 att。

以上是一些常见的 objdump 用法和参数。通过组合使用这些参数，你可以根据具体的需求更深入地了解目标文件的内部结构和代码执行逻辑。