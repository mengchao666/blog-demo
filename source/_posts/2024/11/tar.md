---
title: tar
tags: [Linux工具]
categories: [Linux工具]
permalink: 
poster:
  topic: null
  headline: null
  caption: null
  color: null
date: 2024-11-24 23:20:22
topic: linux_tool
description:
cover:
banner:
references:
---
tar全称是tape archive，初衷是将多个文件写入磁带。首先，需要分清两个概念——打包与压缩。**打包**：将多个文件汇总成一个文件。**压缩**：将一个大文件通过压缩算法变成一个小文件。而tar命令执行的打包流程，真正执行压缩的是使用的压缩算法，比如gzip、bzip2、xz。tar命令在Linux社区内十分受欢迎，其中一个原因就是灵活性强，可以根据需要选择不同的压缩算法。

## 一、常用参数

1. 打包过程

* `-c`或`--create`。创建档案文件（可以理解为压缩包名）
* `-x`或`--extract`。解压（提取）文件
* `-f`或`--file`。指定档案文件，告诉tar命令，后面是文件名
* `-t`或`--list`。列出档案文件的内容
* `-v`或`--verbose`。显示处理文件的详细信息

当多个参数简写在一起的时候，可以只用一个`-`。在实际使用中，最常使用的参数就是`-cvf`，即创建压缩包，并以显示详细处理信息。

2. 压缩过程

* `gzip`：参数`-z`或`--gzip`；文件拓展名：`.tar.gz`或`.tgz`
* `bzip2`：`-j`或`--bzip2`；`.tar.bz2`
* `xz`：`-J`或`--xz`；`.tar.xz`

压缩算法之间的区别：

| 压缩算法   | `gzip`              | `bzip2`         | `xz`           |
| ------------ | --------------- | ---------- | ------------ |
| 参数       | `-z`              | `-j`         | `-J`           |
| 文件拓展名 | `.tar.gz`              | `.tar.bz2`         | `.tar.xz`           |
| 压缩速度   | 快            | 中       | 慢         |
| 解压速度   | 快            | 中       | 中         |
| 压缩比     | 低            | 中       | 高         |
| 资源占用   | 少            | 中       | 高         |
| 适用场景   | 快速压缩/解压 | 高压缩比 | 最大压缩比 |

在日常使用中，使用`gzip`压缩就可以了，虽然压缩比低，但是它十分的快。并且如果被压缩的文件本身就比较小，使用`xz`压缩的结果也不会少太多。因此，日常使用建议`gzip`，既想要速度也想要压缩比建议`bzip2`，超大文件建议`xz`。

## 二、示例

流程相似，只需更换压缩算法的参数。

```shell
# 压缩。压缩包名 + 被压缩的目录或者文件路径
tar -czvf archive_name.tar.gz path_to_compress
# 解压。用什么压缩算法压缩的，就用什么压缩算法解压
tar -xzvf archive_name.tar.gz 
# 解压到当前目录
tar -xzvf archive_name.tar.gz -C path_to_extract 
# 解压至指定目录。 -C （change directory）指出目录地址
```