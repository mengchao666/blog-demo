---
title: Makefile基础语法
tags: [Makefile]
categories: [Makefile]
permalink: 
poster:
  topic: null
  headline: null
  caption: null
  color: null
date: 2024-11-13 21:26:43
topic: project
description:
cover:
banner:
references:
---

## 一、Makefile规则
一个简单的 Makefile 文件包含一系列的“规则”，其样式如下：

```shell
目标(target)…: 依赖(prerequiries)…
<tab>命令(command)
```

目标(target)通常是要生成的文件的名称，可以是可执行文件或OBJ文件， 也可以是一个执行的动作名称，诸如‘ **clean** ’。
依赖是用来产生目标的材料(比如源文件)，一个目标经常有几个依赖。
命令是生成目标时执行的动作，一个规则可以含有几个命令，每个命令占一行。
**注意**：
每个命令行前面必须是一个 Tab 字符，即命令行第一个字符是 Tab。这是容易出错的地方。
通常，如果一个依赖发生了变化，就需要规则调用命令以更新或创建目标。 但是并非所有的目标都有依赖，例如，目标“ **clean** ”的作用是清除文件，它有依赖。

一个 Makefile 文件可以包含规则以外的其他文本，但一个简单的 Makefile 文件仅仅需要包含规则。虽然真正的规则比这里展示的例子复杂，但格式是完全一样的

## 二、make命令介绍
make 命令的使用：

执行 make 命令时，它会去当前目录下查找名为“Makefile”的文件，并根 据它的指示去执行操作，生成第一个目标。
我们可以使用“  **-f** ”选项指定文件，不再使用名为“Makefile”的文件，比 如：

```shell
make -f Makefile.build
```
我们可以使用“  **-C** ”选项指定目录，切换到其他目录里去，比如：
```shell
make -C a/ -f Makefile.build
```
我们可以指定目标，不再默认生成第一个目标：
```shell
make -C a/ -f Makefile.build other_target
```

## 三、Makefile变量介绍

```shell
A = xxx // 延时变量
B ?= xxx // 延时变量，只有第一次定义时赋值才成功；如果曾定义过，此赋值无效
C := xxx // 立即变量
D += yyy // 如果 D 在前面是延时变量，那么现在它还是延时变量；// 如果 D 在前面是立即变量，那么现在它还是立即变量
```

![示例](https://raw.githubusercontent.com/mengchao666/picture/main/blog20241113232347.png)
## 四、Makefile通配符&一些符号
**%通配符**

* %.o：表示所用的.o文件%.
* %.c：表示所有的.c文件

**$@：表示目标**
**$&lt;：表示第一个依赖文件**
**$^：表示所有依赖文件**

![示例代码](https://raw.githubusercontent.com/mengchao666/picture/main/blog20241113232528.png)
![结果](https://raw.githubusercontent.com/mengchao666/picture/main/blog20241113232545.png)

## 五、Makefile假想目标
我们的 Makefile 中有这样的目标：
```Makefile
clean:
  rm -f $(shell find -name "\*.o")
  rm -f $(TARGET)
```

如果当前目录下恰好有名为“clean”的文件，那么执行“ **make clean** ”时它 就不会执行那些删除命令。
这时我们需要把“ **clean** ”这个目标，设置为“假想目标”，这样可以确保执行“ **make clean** ”时那些删除命令肯定可以得到执行。

使用下面的语句把“clean”设置为假想目标：
```Makefile
.PHONY : clean
```

![](https://raw.githubusercontent.com/mengchao666/picture/main/blog20241113232733.png)
![](https://raw.githubusercontent.com/mengchao666/picture/main/blog20241113232751.png)

## 六、Makefile常用函数
### 1.$(foreach var,list,text)

简单地说，就是 for each var in list, change it to text。对 list 中的每一个 元素，取出来赋给 var，然后把 var 改为 text 所描述的形式。

### 2.$(wildcard pattern)
pattern 所列出的文件是否存在，把存在的文件都列出来。

### 3.$(filter pattern...,text)
把 text 中符合 pattern 格式的内容，filter(过滤)出来、留下来。

### 4.$(filter-out pattern...,text)
把 text 中符合 pattern 格式的内容，filter-out(过滤)出来、扔掉。

### 5.$(patsubst pattern,replacement,text)
寻找' **text** '中符合格式' pattern '的字，用' r**eplacement** '替换它们。

' **pattern** '和' **replacement** '中可以使用通配符。
![代码示例](https://raw.githubusercontent.com/mengchao666/picture/main/blog20241113234721.png)
## 七、Makefile的例子，包含子目录

## 八、Makefile中的EXTRA_CFLAGS

EXTRA_CFLAGS是Makefile中预定义的一个变量，作为CFLGAS，在make时可以传递给gcc一些编译选项等，如—O2
### EXTRA_CFLAGS += -D等价于gcc -D，相当于在源代码中定义一个宏
假如定义一个宏CONFIG_DEBUG
在.c里面定义为：#define CONFIG_DEBUG
在makefile里定义为: CONFIG_DEBUG=y

假如说我们想在makefile里为.c文件进入一个宏定义，就用EXTRA_CFLAGS += DCONFIG_DEBUG( 等价于在.c文件里定义#define CONFIG_DEBUG)

这时CONFIG_DEBUG=y与EXTRA_CFLAGS += DCONFIG_DEBUG的区别应该你已经看出来的，前者是对makefile编译时用的，比如说obj-(CONFIG_DEBUG) += test.o,而后者则是对.c源文件里的用的

## 九、Makefile编译内核驱动ko
内核源代码中obj-m表示以模块ko的方式编译
obj-y表示将源代码编译到内核源码中
在工作的过程中，经常需要编译一些Ko模块，如果是单个的c文件编译直接在内核源码里面 obj-y=xxx.o就好
如果这个ko文件需要多个c文件共同编译生成的话，最好以如下的模板来完成编译较好

```shell
#首先指定好编译链工具
CROSS_COMPILE=/opt/gcc-linaro-7.3.1-2018.05-x86_64_aarch64-linux-gnu/bin/aarch64-linux-gnu-
#指定用哪个内核去编译
KDIR=/mnt/nfsroot/zhengshuai.zhu/IPCSDK/ipc-sdk-full-release/kernel-4.19

#目标是编译成一个test.ko文件
obj-m += test.o
#test.o文件由当前目录下n个c文件编译成
test-objs += ./*.o
#包含当前目录下的头文件
INCLUDE_DIRS := $(addprefix -I,$(shell find ../ -type d ))

#包含当前目录下，内核目录下的头文件
ccflags-y:= -I$(_KDIR)/include/linux/ -I$(PWD)/platform/
#忽略一些编译警告，类如什么变量未使用
ccflags-y += -Wno-declaration-after-statement

#添加c文件中的环境变量，比如在代码中会有
#ifdef CONFIG_ANDROID 
#xxxx
#endif
ifeq ($(SYSTEM_VERSION),)
    ccflags-y += -DCONFIG_LINUX_OS
else
    ccflags-y += -DCONFIG_ANDROID
endif

all:
	make ARCH=${ARCH} -C $(KDIR) M=$(PWD) modules

clean:
	make ARCH=${ARCH} -C $(KDIR) M=$(PWD) clean
```

`make ARCH=${ARCH} -C ( K D I R ) M = (KDIR) M=(KDIR)M=(PWD) modules`
如何理解这句话?
-C的选项可以理解为:
进入所指定的位置，$(KDIR)，也就是内核目录中，目的是什么？ 去读取内核目录顶层的Makefile文件，相当于编译的时候 选择一个内核，我要用这个内核去编译。
因为你这个目录没有被配置到kernel config里面去，也就是说没有指定用哪个内核版本，有了 -C $(KDIR),就相当于选了内核，如果你选择了kernel-4.19目录下，或者 kernel-5.0目录下，

M=的选项可以理解为:
当我选好内核版本后，我用这个 版本的内核 要去编译哪个目录，然后进入$(PWD)目录去编译当前指定的文件，将其编译成ko文件
