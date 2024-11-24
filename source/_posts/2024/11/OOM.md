---
title: OOM
tags: [Linux]
categories: [Linux]
permalink: 
poster:
  topic: null
  headline: null
  caption: null
  color: null
date: 2024-11-24 23:37:20
topic: linux
description:
cover:
banner:
references:
---
在Linux操作系统中，OOM（Out of Memory）指的是系统可用内存耗尽，无法再为任何进程分配所需的内存，从而导致系统必须采取某些极端措施的情况。当系统面临OOM时，可能会选择杀死某些进程以释放内存，这是通过Linux内核的OOM Killer机制实现的。

### OOM发生的原因：

1. **物理内存不足**：实际物理内存资源耗尽，无法满足所有进程的内存需求，尤其是当大量进程同时运行且内存占用较大时更容易出现。
2. **交换空间不足**：即使有交换分区（Swap），但当系统试图将物理内存中的页换出到交换空间时，发现交换空间也已满，无法继续进行内存交换。
3. **内存泄漏**：应用程序存在内存泄漏问题，随着时间推移不断消耗内存，直至耗尽整个系统资源。
4. **一次性加载大量数据**：某个进程瞬间请求大量内存，超出了系统所能提供的范围。
5. **内存限制**：在容器环境下，单个容器可能存在严格的内存限制，超出限制后也会触发OOM。