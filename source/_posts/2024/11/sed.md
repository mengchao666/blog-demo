---
title: sed
tags: [Linux工具]
categories: [Linux工具]
permalink: 
poster:
  topic: null
  headline: null
  caption: null
  color: null
date: 2024-11-24 23:21:02
topic: linux_tool
description:
cover:
banner:
references:
---
https://blog.csdn.net/2301_78315274/article/details/133880462

1、将每行第一个111替换为AAA

```c
sed -i "s/111/AAA/" a.txt
```

-i作用会修改源文件，如这里的a.txt，如果不加-i就不会修改，只是命令回显改变了

2、替换所有的111为AAA

```c
1. sed -i "s/111/AAA/g" a.txt
```

3、替换第一到四行的所有111为AAA

```c
sed "1,4s/111/AAA/g" a.txt
```