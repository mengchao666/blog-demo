---
title: mmap
tags: [C语言]
categories: [C语言]
permalink: 
poster:
  topic: null
  headline: null
  caption: null
  color: null
date: 2024-11-25 22:49:19
topic: c
description:
cover:
banner:
references:
---
mmap 即 memory map，也就是内存映射。mmap 是一种内存映射文件的方法，即将一个文件或者其它对象映射到进程的地址空间，实现文件磁盘地址和进程虚拟地址空间中一段虚拟地址的一一对映关系。实现这样的映射关系后，进程就可以采用指针的方式读写操作这一段内存，而系统会自动回写脏页面到对应的文件磁盘上，即完成了对文件的操作而不必再调用 read、write 等系统调用函数。相反，内核空间对这段区域的修改也直接反映用户空间，从而可以实现不同进程间的文件共享。

![](https://raw.githubusercontent.com/mengchao666/picture/main/blog5175f10387866d6173bea7fbe89c4eeb.webp)

mmap原型

```c
#include <sys/mman.h>
void *mmap(void *start, size_t length, int prot, int flags, int fd, off_t offset)
```

==必须注意这里的映射长度length必须是4K整数倍==

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/mman.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>
 
int main() {
	int my_data;
    int fd = open("/dev/mem", O_RDWR | O_SYNC);
    if (fd < 0) {
        perror("open");
        return 1;
    }
 
    unsigned long addr = 0x80000000; // 假设我们要访问的物理地址
    unsigned long map_length = 0x1000; // 映射的长度为4KB
 
    // 映射内存
    void *ptr = mmap(NULL, map_length, PROT_READ | PROT_WRITE, MAP_SHARED, fd, addr & ~(getpagesize() - 1)); // 获取页面对齐基地址，必须以4K对齐
    if (ptr == MAP_FAILED) {
        perror("mmap");
        close(fd);
        return 1;
    }

	my_data = *(int *)(ptr + addr & (getpagesize() - 1));

    // 解除内存映射
    if (munmap(ptr, map_length) < 0) {
        perror("munmap");
        close(fd);
        return 1;
    }
 
    // 关闭文件描述符
    close(fd);
    return 0;
}
```