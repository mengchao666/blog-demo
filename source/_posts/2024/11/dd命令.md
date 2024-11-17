---
title: dd命令
tags: []
categories: []
permalink: 
poster:
  topic: null
  headline: null
  caption: null
  color: null
date: 2024-11-17 14:15:34
topic: linux_tool
description:
cover:
banner:
references:
---

> dd命令可以从一个文件或设备向另一个文件或设备进行复制

## 一、dd命令常用语法

```
dd if=input_file of=output_file [options]
```

options是一些可选参数，if和of是必须有的

* if 表示输入文件
* of 表示输出文件
* bs：设置每次读取和写入的块大小（单位为字节或者是可以添加的后缀，如b、k、m等），默认为512字节。
* count：设置要复制的块数。
* iflag：设置输入选项，常用的选项有direct（绕过缓存直接读取）和sync（同步数据到磁盘）。
* oflag：设置输出选项，常用的选项有direct（绕过缓存直接写入）和sync（同步数据到磁盘）。
* skip=xxx 是在备份时对if 后面的部分也就是原文件跳过多少块再开始备份；
* seek=xxx则是在备份时对of 后面的部分也就是目标文件跳过多少块再开始写

## 二、dd常用命令-读磁盘

```c
dd if=/dev/sde1 of=tee-test.img bs=1M skip=4 count=16
```

含义为从/dev/sde1设备起始位置，跳过4M，读取16M内容到tee-test.img中
想要查看tee-test.img，可以使用hexdump命令

```c
hexdump -C tee-test.img
```

只查看部分内容，例如前100个字节：

```c
hexdump -C -n 100 tee-test.img
```

## 三、dd常用命令-写磁盘

```
dd if=/dev/zero of=/dev/sda bs=1k seek=8224 count=32
```

含义为从/dev/zero中也就是将后面的of目标文件写0，将/dev/sda设备的起始地址，跳过8224KB后，连续写入32KB 0数据

## 四、通过fdisk -l确认分区的实际起始地址

使用`fdisk -l`显示磁盘分区情况

```
# fdisk -l

Disk /dev/sde: 19.16 GB, 20577255424 bytes, 5023744 sectors
Sector size (logical/physical): 4096 bytes / 4096 bytes

  Device  Start     End
/dev/sde1   256     131327

```

这里显示sde1的起始地址为256，那么实际地址为：
256 ✖️ 4096 = 1048576 = 1024 ✖️ 1024 = 1MB，所以sde1的位置为/dev/sde的起始地址偏移1M，其实起始1M，存放的应该是磁盘分区信息