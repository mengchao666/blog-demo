---
title: objcopy
tags: [Linux工具]
categories: [Linux工具]
permalink: 
poster:
  topic: null
  headline: null
  caption: null
  color: null
date: 2024-11-24 23:18:36
topic: linux_tool
description:
cover:
banner:
references:
---
objcopy转换elf文件为bin文件，以下是一个将boot.elf转为boot.bin的命令

```C
arm-linux-objcopy -O binary -R .note -R .comment -S boot.elf boot.bin
```

* 使用 -O binary (或--out-target=binary) 输出为原始的二进制文件
* 使用 -R .note (或--remove-section) 输出文件中不要.note这个section，缩小了文件尺寸
* 使用 -S (或 --strip-all) 输出文件中不要重定位信息和符号信息，缩小了文件尺寸