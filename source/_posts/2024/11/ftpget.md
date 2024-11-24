---
title: ftpget
tags: [Linux工具]
categories: [Linux工具]
permalink: 
poster:
  topic: null
  headline: null
  caption: null
  color: null
date: 2024-11-24 23:31:27
topic: linux_tool
description:
cover:
banner:
references:
---
ftpget -u  username -p passwd IP  source  target

ftpput -u  username -p passwd IP  target  source

举个例子：

```sh
ftpput -u zhangsan -p 000000 192.168.10.10 target.txt source.txt   
// 将本地的 source.txt 文件传输到 192.168.10.10 /home/zhangsan/ 目录下，并以target.txt 保存
```