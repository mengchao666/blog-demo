---
title: scp
tags: [Linux工具]
categories: [Linux工具]
permalink: 
poster:
  topic: null
  headline: null
  caption: null
  color: null
date: 2024-11-24 23:30:53
topic: linux_tool
description:
cover:
banner:
references:
---
## 一、从本地复制到远程主机

命令格式如下：

scp /path/to/local/file.txt user@remote_host:/path/on/remote/

这会将本地的 `file.txt` 文件复制到远程主机 `remote_host` 的 `/path/on/remote/` 目录下。

### 二、从远程主机复制到本地

（1）命令解释
命令格式如下：scp user@remote_host:/path/on/remote/file.txt /path/to/local/

这会将远程主机 `remote_host` 的 `/path/on/remote/file.txt` 文件复制到本地的 `/path/to/local/` 目录下。

（2）实际操作
实操命令如下：
scp root@192.168.1.109:/home/DataBaseMysql.zip ./

这会将远程主机 `192.168.1.109` 的/home/DataBaseMysql.zip 文件复制到本地的当前目录下

### 三、递归复制目录

实操命令如下：scp -r user@remote_host:/path/on/remote/directory /path/to/local/

这会将远程主机 `remote_host` 的 `/path/on/remote/directory` 目录及其所有内容复制到本地的 `/path/to/local/` 目录下。

### 四、指定 SSH 端口

如果远程主机的 SSH 端口不是默认的 22，可以使用 `-P` 选项指定端口：

scp -P 2222 user@remote_host:/path/on/remote/file.txt /path/to/local/