---
title: tcpdump
tags: [Linux工具]
categories: [Linux工具]
permalink: 
poster:
  topic: null
  headline: null
  caption: null
  color: null
date: 2024-11-24 23:21:54
topic: linux_tool
description:
cover:
banner:
references:
---
tmpdump用于抓包，一个例子：

```c
tcpdump -i any host 192.168.x.x -s0 -vvv -w 1.cap
```

* -i any 任何网络
* -s0 防止截断
* -w写入文件
* -vvv详细的信息

最终得到一个名为1.cap的文件，可以使用wireshark工具打开