---
title: module_init宏详解&insmod流程&THIS_MODULE
tags: [kernel]
categories: [kernel]
permalink: 
poster:
  topic: null
  headline: null
  caption: null
  color: null
date: 2024-11-25 23:19:20
topic: linux_kernel
description:
cover:
banner:
references:
---
一个模块可以编译进内核，和内核一起打包，也可以作为一个单独模块，单独加载，如ko，本节以ko为例

以最简单的例子讲解：

```c
// //helloworld.c

#include <linux/kernel.h>
#include <linux/module.h>

//内核模块初始化函数
static int __init hello_init(void)
{
	printk("Hello World\n");
	return 0;
}

//内核模块退出函数
static void __exit hello_exit(void)
{
	printk("exit\n");
}

module_init(hello_init);
module_exit(hello_exit);

MODULE_LICENSE("GPL");
```

## 一、一个特殊的段.gnu.linkonce.this_module section(THIS_MODULE定义)

当我们编译生成.ko模块时，会在该目录下产生这样一个附加文件helloworld.mod.c，这个文件内容如下：
其实THIS_MODULE在内核的定义就是\_\_this_module，通过这个宏就能拿到本模块的一些信息

```c
// helloworld.mod.c

__visible struct module __this_module
__section(".gnu.linkonce.this_module") = {
	.name = KBUILD_MODNAME,
	.init = init_module,
#ifdef CONFIG_MODULE_UNLOAD
	.exit = cleanup_module,
#endif
	.arch = MODULE_ARCH_INIT,
};
```

这个后缀为.mod.c的文件会初始化模块 struct module 的这几个成员：

```c
struct module {
	......
	/* Unique handle for this module */
	char name[MODULE_NAME_LEN];

	/* Startup function. */
	int (*init)(void);

	/* Destruction function. */
	void (*exit)(void);

	/* Arch-specific module values */
	struct mod_arch_specific arch;
	......
}
```

通过__section(“.gnu.linkonce.this_module”)我们可以知道将该struct module实例添加到内核模块ELF文件中“.gnu.linkonce.this_module”section中。

```sh
readelf -S helloworld.ko
```

![](https://raw.githubusercontent.com/mengchao666/picture/main/blog1ab338456ff54936a8c8e4851bd837f8.png)

## 二、初始化和清理函数

### 初始化和清理函数

在这里我们发现.mod.c后缀文件的模块初始化函数和清理函数名字是init_module和cleanup_module，而我们编写的内核模块程序中是module_init和module_exit：

```c
.init = init_module,
.exit = cleanup_module,
```

```c
module_init(hello_init);
module_exit(hello_exit);
```

接下来咱们一起看看module_init源码实现细节：

## 三、module_init的定义

其实module_init在内核中<linux/module.h>中有两个定义，通过MODULE宏来区分，在咱们编译ko时，使用的命令通常如下：

```c
make -C $(KDIR) M=$(PWD) modules
```

这时会进入内核源码树的顶层Makefile下查找modules目标协助编译，MODULE宏的定义是由内核顶层的Makefile和script/Makefile.lib配合实现的

Makefile

```c
KBUILD_AFLAGS_MODULE  := -DMODULE
KBUILD_CFLAGS_MODULE  := -DMODULE
```

scripts/Makefile.lib

```c
part-of-module = $(if $(filter $(basename $@).o, $(real-obj-m)),y)
quiet_modtag = $(if $(part-of-module),[M],   )
modkern_cflags =                                          \
       $(if $(part-of-module),                           \
                $(KBUILD_CFLAGS_MODULE) $(CFLAGS_MODULE), \
                $(KBUILD_CFLAGS_KERNEL) $(CFLAGS_KERNEL) $(modfile_flags))
```

$@ 表示正在编译的目标，如果是 module 的一部分，则使用 KBUILD_CFLAGS_MODULE 作为 cflags ，即 -DMODULE 被引入 gcc 命令行

此时module_init实现如下：

```c
// linux-4.10.1/include/linux/module.h

/* Each module must use one module_init(). */
#define module_init(initfn)					\
	static inline initcall_t __inittest(void)		\
	{ return initfn; }					\
	int init_module(void) __attribute__((alias(#initfn)));

/* This is only required if you want to be unloadable. */
#define module_exit(exitfn)					\
	static inline exitcall_t __exittest(void)		\
	{ return exitfn; }					\
	void cleanup_module(void) __attribute__((alias(#exitfn)));

```

这使用了gcc提供的别名属性（**attribute**(alias)）：<br />将init_module函数的别名设置为initfn，而initfn就是我们模块程序中的初始化函数。<br />将cleanup_module的别名设置为exitfn，而exitfn就是我们模块程序的清理函数。

## 四、\_\_init函数和__exit函数

定义如下：

```c
#define __init		__section(.init.text) __cold notrace __latent_entropy
#define __exit      __section(.exit.text) __exitused __cold notrace
```

这两个前缀是为了将初始化函数和清理函数放置在ko二进制文件正确的section中。

![](https://raw.githubusercontent.com/mengchao666/picture/main/blog92d67e5ad6f04ae9a35d6b000d7762db.png)

## 五、insmod流程

insmod对应的是busybox的一个命令，可以看busybox中的insmod_main函数中的具体实现，这里简单描述一下：

insmod将ko文件使用mmap映射，获取到地址，然后通过系统调用SYSCALL_DEFINE3进入内核

```c
SYSCALL_DEFINE3(init_module)
	-->load_module()
		-->layout_and_allocate()
			-->setup_load_info()
```

load_module中会把.gnu.linkonce.this_module section中的内容初始化给 struct module，然后通过一些列操作，重定位等，获取到.init_text中的位置，最后调用进去，也就是调用到了module_init中的函数，也就是本例中的hello_init。至此ko加载成功，hello_init函数执行