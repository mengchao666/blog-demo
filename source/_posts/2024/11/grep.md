---
title: grep
tags: [Linux工具]
categories: [Linux工具]
permalink: 
poster:
  topic: null
  headline: null
  caption: null
  color: null
date: 2024-11-24 23:29:25
topic: linux_tool
description:
cover:
banner:
references:
---
文本搜索工具，根据用户指定的“模式”（过滤条件），对目标文本逐行进行匹配，并打印输出匹配到的行。

完整语法：

```c
grep   [options]   [pattern]   file
 
命令    参数         匹配模式      文件数据
```

常用参数：

| 常用参数 | 描述                     |
| ---------- | -------------------------- |
| -i       | 忽略大小写               |
| -n       | 显示匹配行与行号         |
| -r       | 递归查找子目录           |
| -v       | 显示不能被匹配到的字符串 |

常用正则表达式

表达式	解释

* ^	用于模式最左侧，如 “^yu” 即匹配以yu开头的单词
* $	用于模式最右侧，如 “yu$” 即匹配以yu结尾的单词
* ^$	组合符，表示空行
* .	匹配任意一个且只有一个字符，不能匹配空行
* |	使用egrep命令
*  

  * 重匹配前一个字符连续出现0次或1次以上
* .*	匹配任意字符
* ^.*	组合符，匹配任意多个字符开头的内容
* .*$	组合符，匹配任意多个字符结尾的内容
* [abc]	匹配 [] 内集合中的任意一个字符，a或b或c，也可以写成 [ac]
* [^abc]	匹配除了 ^后面的任意一个字符，a或b或c，[]内 ^ 表示取反操作

常用：

```c
grep -nr "xxx" .
```