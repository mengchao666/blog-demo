---
title: top
tags: [Linux工具]
categories: [Linux工具]
permalink: 
poster:
  topic: null
  headline: null
  caption: null
  color: null
date: 2024-11-24 23:24:01
topic: linux_tool
description:
cover:
banner:
references:
---
```Bash
ps -ef | grep xxx //xxx为进程名字
```

通过ps找到进程号，通过如下命令可以查看该进程下的所有**线程**CPU利用率，注意这里是该进程PID下对应的所有线程

```Bash
top -H -p pid
```

例如进程pid为5810，则命令为：

```C
top -H -p 5810
```

## 一、top命令的使用帮助

### 1、top命令的选项

top命令的使用方法：top [-d number] | top [-bnp]

|            |                    |
| ------------ | -------------------- |
| 选项       | 解析               |
| -b         | 以批处理模式操作； |
| -c         | 显示完整的治命令； |
| -d         | 屏幕刷新间隔时间； |
| -I         | 忽略失效过程；     |
| -s         | 保密模式；         |
| -S         | 累积模式；         |
| -i<时间>   | 设置间隔时间；     |
| -u<用户名> | 指定用户名；       |
| -p<进程号> | 指定进程；         |
| -n<次数>   | 循环显示的次数。   |

## 二、top命令的交换命令

在top命令执行过程中可以使用的一些交互命令。这些命令都是单字母的，如果在命令行中使用了-s选项， 其中一些命令可能会被屏蔽。

**也就是在top命令运行过程中，可以按下如下案件，会按照相应指令显示**

```Bash
h：显示帮助画面，给出一些简短的命令总结说明；
k：终止一个进程；
i：忽略闲置和僵死进程，这是一个开关式命令；
q：退出程序；
r：重新安排一个进程的优先级别；
S：切换到累计模式；
s：改变两次刷新之间的延迟时间（单位为s），如果有小数，就换算成ms。输入0值则系统将不断刷新，默认值是5s；
f或者F：从当前显示中添加或者删除项目；
o或者O：改变显示项目的顺序；
l：切换显示平均负载和启动时间信息；
m：切换显示内存信息；
t：切换显示进程和CPU状态信息；
c：切换显示命令名称和完整命令行；
M：以内存的使用资源排序显示；
P：根据CPU使用百分比大小进行排序；
T：根据时间/累计时间进行排序；
w：将当前设置写入~/.toprc文件中。
```

## 三、top显示信息解释

### 1、top的第一行解释

在命令行输入top，进入系统监控信息的交互界面，第一行解释如下：

```C
10:40:53          表示当前时间
up  7:09         系统运行时间，格式为时：分。
3 users      当前登录用户数
load average: 0.05, 0.03, 0.05        系统负载，即任务队列的平均长度。 三个数值分别为 1分钟、5分钟、15分钟前到现在的平均值。
```

![](https://hwwyaazvtut.feishu.cn/space/api/box/stream/download/asynccode/?code=MTVlZDRhZGZiM2ZmODg1N2E5YjFiZDAwMjkwNTkyYzVfNElvVk1FM2Z0bThxa3VlSmVBdU5MajJOeUpXbFVSc0lfVG9rZW46WmFGNWI1c1N4b0NMNUV4aFhZU2NVdERDbmZlXzE3MjEzMTI0NDQ6MTcyMTMxNjA0NF9WNA)

### 2、top的第二、三行信息解释

在命令行输入top，进入系统监控信息的交互界面，第2、3行为进程和CPU的信息，当有多个CPU时，这些内容可能会超过两行，

第二行解释如下：

```C
216 total          进程总数
1 running          正在运行的进程数
215 sleeping       睡眠的进程数
0 stopped          停止的进程数
0 zombie          僵尸进程数
0.0 us              用户空间占用CPU百分比
0.1 sy              内核空间占用CPU百分比
0.0 ni              用户进程空间内改变过优先级的进程占用CPU百分比
99.9 id              空闲CPU百分比
0.0 wa              等待输入输出的CPU时间百分比
0.0 hi              硬中断（Hardware IRQ）占用CPU的百分比
0.0 si              软中断（Software Interrupts）占用CPU的百分比
0.0 st              虚拟CPU等待实际CPU的时间的百分比。
```

### 3、top的第四、五行信息解释

第四行及第五行主要显示系统的内存信息。

```C
KiB Mem: 12119056 tota         物理内存总量
10016948 free                 空闲内存总量
923252 used                    使用的物理内存总量
1178856 buff/cache             用作内核缓存的内存量
KiB Swap: 2093052 total         交换区总量
267544 used                     使用的交换区总量
2093052 free                 空闲交换区总量
0 used                         缓冲的交换区总量。
10742188 avail Mem             代表可用于进程下一次分配的物理内存数量
```

### 4、top的进程信息

top命令的交换界面主要区域，监控系统进程的实时状态信息。

```C
PID            进程id
USER    进程所有者的用户名
PR            优先级
NI            nice值，负值表示高优先级，正值表示低优先级。
VIRT    进程使用的虚拟内存总量，单位kb。VIRT=SWAP+RES
RES            进程使用的、未被换出的物理内存大小，单位kb。RES=CODE+DATA
SHR            共享内存大小，单位kb
S            进程状态。D=不可中断的睡眠状态 R=运行 S=睡眠 T=跟踪/停止 Z=僵尸进程
%CPU    上次更新到现在的CPU时间占用百分比
%MEM    进程使用的物理内存百分比
TIME+   进程使用的CPU时间总计，单位1/100秒
COMMAND 命令名/命令行
```

其余监控项解释

```Bash
PPID        父进程id
RUSER        Real user name
UID            进程所有者的用户id
GROUP   进程所有者的组名
TTY            启动进程的终端名。不是从终端启动的进程则显示为 ?
P            最后使用的CPU，仅在多CPU环境下有意义
TIME        进程使用的CPU时间总计，单位秒
SWAP        进程使用的虚拟内存中，被换出的大小，单位kb
CODE        可执行代码占用的物理内存大小，单位kb
DATA        可执行代码以外的部分(数据段+栈)占用的物理内存大小，单位kb
nFLT        页面错误次数
nDRT        最后一次写入到现在，被修改过的页面数。
WCHAN        若该进程在睡眠，则显示睡眠中的系统函数名
Flags        任务标志
```

## 四、top命令的基本使用

1、查看当前系统cpu占用最高的进程

进入top交互界面后，按P键对CPU负载的进程进行排列。

2、查看当前系统内存使用最高的进程

进入top交互界面后，按M键对CPU负载的进程进行排列。

3、对排序的列进行高亮显示

敲击键盘‘x’（打开/关闭排序列的加亮效果）

4、对运行的进程进行高亮显示

敲击键盘‘b’（打开关闭加亮效果），对运行的进程进行高亮显示