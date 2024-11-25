---
title: Linux内核的initcall机制
tags: [kernel]
categories: [kernel]
permalink: 
poster:
  topic: null
  headline: null
  caption: null
  color: null
date: 2024-11-25 23:17:08
topic: linux_kernel
description:
cover:
banner:
references:
---
从根文件系统的挂载说起吧，使能了initd小文件系统的OS，最终会调用到populate_rootfs执行解压挂载的操作，搜索populate最终看到是rootfs_initcall被执行到的，那就从此处开始说起吧，看看内核是如何执行到这个函数的

## 一、initcall函数调用流程

调用链如下：

```c
init/main.c
调用链如下：
 
start_kernel()
    ---->arch_call_rest_init()
        ---->rest_init()
            ---->kernel_init()
                ---->kernel_init_freeable()
                    ---->do_basic_setup()
                        ---->do_initcalls()
```

```c
 
static initcall_entry_t *initcall_levels[] __initdata = {
	__initcall0_start,
	__initcall1_start,
	__initcall2_start,
	__initcall3_start,
	__initcall4_start,
	__initcall5_start,
	__initcall6_start,
	__initcall7_start,
	__initcall_end,
};
 
static void __init do_initcalls(void)
{
	int level;
 
	for (level = 0; level < ARRAY_SIZE(initcall_levels) - 1; level++)
		do_initcall_level(level);
}
```

for 循环针对一个指针数组轮询，该数组是一个静态的 initcall_entry_t 类型，这些数据都存放在 __initdata 段。

指针数组的类型为 initcall_entry_t，是 initcall_t 的另一种叫法

继续来看下这个指针数组中的元素：\_\_initcall0_start ~ \_\_initcall_end，而这些元素的值在本c文件中已经声明：

```c
init/main.c
 
extern initcall_entry_t __initcall_start[];
extern initcall_entry_t __initcall0_start[];
extern initcall_entry_t __initcall1_start[];
extern initcall_entry_t __initcall2_start[];
extern initcall_entry_t __initcall3_start[];
extern initcall_entry_t __initcall4_start[];
extern initcall_entry_t __initcall5_start[];
extern initcall_entry_t __initcall6_start[];
extern initcall_entry_t __initcall7_start[];
extern initcall_entry_t __initcall_end[];
```

不难看出，数组 initcall_levels 中元素存放的是这些函数指针数组的首地址。

那么这些实际的指针数组是在哪里呢？从上文得知，initcall 函数都会被定义成 static initcall_t 类型，并且保存在 .initcall##level##.init 段中，那么 initcall_levels 与其是怎么关联的呢？

答案在 vmlinux.lds.h 中。

## 二、vmlinux.lds.h

```c
include/asm-generic/vmlinux.lds.h
 
#define INIT_CALLS_LEVEL(level)						\
		__initcall##level##_start = .;				\
		KEEP(*(.initcall##level##.init))			\
		KEEP(*(.initcall##level##s.init))			\
#define INIT_CALLS							\
		__initcall_start = .;					\
		KEEP(*(.initcallearly.init))				\
		INIT_CALLS_LEVEL(0)					\
		INIT_CALLS_LEVEL(1)					\
		INIT_CALLS_LEVEL(2)					\
		INIT_CALLS_LEVEL(3)					\
		INIT_CALLS_LEVEL(4)					\
		INIT_CALLS_LEVEL(5)					\
		INIT_CALLS_LEVEL(rootfs)				\
		INIT_CALLS_LEVEL(6)					\
		INIT_CALLS_LEVEL(7)					\
		__initcall_end = .;
```

在这里首先定义了__initcall_start，将其关联到".initcallearly.init"段。

然后对每个level定义了INIT_CALLS_LEVEL(level)，将INIT_CALLS_LEVEL(level)展开之后的结果是定义\_\_initcall##level##_start，并将\_\_initcall##level##_start关联到".initcall##level##.init"段和".initcall##level##s.init"段。

```c
	__initcall_start = .;           \
	*(.initcallearly.init)          \
	__initcall0_start = .;          \
	*(.initcall0.init)              \
	*(.initcall0s.init)             \
	// 省略1、2、3、4、5
	__initcallrootfs_start = .;     \
	*(.initcallrootfs.init)         \
	*(.initcallrootfss.init)            \
	__initcall6_start = .;          \
	*(.initcall6.init)              \
	*(.initcall6s.init)             \
	__initcall7_start = .;          \
	*(.initcall7.init)              \
	*(.initcall7s.init)             \
	__initcall_end = .;
```

上面这些代码段最终在kernel.img中按先后顺序组织，也就决定了位于其中的一些函数的执行先后顺序。

在 kernel_init() 初始化完成后，会调用 free_initmem() 对内核 init 段的内存进行释放处理。即 .init 或者 .initcalls 段在内核启动完毕后，这个段中的内存会被释放掉。

## 三、do_initcall_level

do_initcall_level函数定义如下：

```c
init/main.c
 
static void __init do_initcall_level(int level)
{
	initcall_entry_t *fn;
 
	strcpy(initcall_command_line, saved_command_line);
	parse_args(initcall_level_names[level],
		   initcall_command_line, __start___param,
		   __stop___param - __start___param,
		   level, level,
		   NULL, &repair_env_string);
 
	trace_initcall_level(initcall_level_names[level]);
	for (fn = initcall_levels[level]; fn < initcall_levels[level+1]; fn++)
		do_one_initcall(initcall_from_entry(fn));
}
```

do_initcall_level() 函数的参数 level 是之前是initcall_levels 数组的索引，从第0个开始。

这里再用一个 for 循环，跳到 initcall_levels 内部元素 (函数指针数组)进行轮询，fn 初始值为函数指针数组的起始地址，后面 fn++ 相当于函数指针 +1，即跳到下一个函数指针。

即for 循环中，根据传入的 level，确定需要轮询的 .initcall##level##.init 段的所有函数指针，一直到下一个 .intcall##(level+1)##.init 段。

另外，需要注意 do_one_initcall() 的参数就是获取函数指针的内容，而这个内容就是注册进来的 initcall 的实际初始化函数。

如上面的举例：

```c
rootfs_initcall(populate_rootfs);
```

这里最终就看成调用 do_one_initcall(populate_rootfs)；

```c
init/main.c
 
int __init_or_module do_one_initcall(initcall_t fn)
{
	...

	ret = fn();
	...
	return ret;
}
```

## 四、initcall源码

上文提到过 Linux 对驱动程序提供静态编译进内核和动态加载两种方式，Linux 的 initcall 机制也是根据静态编译和动态加载的两种方式选择不同的编译、运行流程。

```c
include/linux/init.h
 
#ifndef MODULE
 
... //静态加载
 
#else
 
... //动态加载
 
#endif
```

MODULE 是在编译的时候，通过编译器参数来传入。例如，在编译 ko 时会使用如下两个编译选项，如果是链接到内核，则不会使用：

```c
//Makefile
 
KBUILD_AFLAGS_MODULE  := -DMODULE
KBUILD_CFLAGS_MODULE  := -DMODULE
```

通过 MODULE 的配置，选择静态编译还是动态加载。

本文将只分析静态编译流程

内核中定义的initcall如下：

```c
include/linux/init.h
 
 
/*
 * Early initcalls run before initializing SMP.
 *
 * Only for built-in code, not modules.
 */
#define early_initcall(fn)		__define_initcall(fn, early)
 
/*
 * A "pure" initcall has no dependencies on anything else, and purely
 * initializes variables that couldn't be statically initialized.
 *
 * This only exists for built-in code, not for modules.
 * Keep main.c:initcall_level_names[] in sync.
 */
#define pure_initcall(fn)		__define_initcall(fn, 0)
 
#define core_initcall(fn)		__define_initcall(fn, 1)
#define core_initcall_sync(fn)		__define_initcall(fn, 1s)
#define postcore_initcall(fn)		__define_initcall(fn, 2)
#define postcore_initcall_sync(fn)	__define_initcall(fn, 2s)
#define arch_initcall(fn)		__define_initcall(fn, 3)
#define arch_initcall_sync(fn)		__define_initcall(fn, 3s)
#define subsys_initcall(fn)		__define_initcall(fn, 4)
#define subsys_initcall_sync(fn)	__define_initcall(fn, 4s)
#define fs_initcall(fn)			__define_initcall(fn, 5)
#define fs_initcall_sync(fn)		__define_initcall(fn, 5s)
#define rootfs_initcall(fn)		__define_initcall(fn, rootfs)
#define device_initcall(fn)		__define_initcall(fn, 6)
#define device_initcall_sync(fn)	__define_initcall(fn, 6s)
#define late_initcall(fn)		__define_initcall(fn, 7)
#define late_initcall_sync(fn)		__define_initcall(fn, 7s)
 
#define __initcall(fn) device_initcall(fn)
 
#define __exitcall(fn)						\
	static exitcall_t __exitcall_##fn __exit_call = fn
 
#define console_initcall(fn)	___define_initcall(fn, con, .con_initcall)
```

对于静态编译 initcall 接口如上，其中 **pure_initcall()**  只能在静态编译中存在。

当然，对于静态编译的驱动也可以调佣 **module_init()**  接口：

```c
include/linux/module.h
 
#define module_init(x)	__initcall(x);
 
#define module_exit(x)	__exitcall(x);
```

> **module_init在内核中是有两种定义的，module_init定义在&lt;linux/module.h&gt;中，当包含此头文件的代码中，没有定义MODULE宏时，module_init定义为initcall形式**

## 五、initcall级别

```c
early-->
0-->0s-->
1-->1s-->
2-->2s-->
3-->3s-->
4-->4s-->
5-->5s-->
rootfs-->
6-->6s-->
7-->7s-->
console
```

\_\_define_initcall实现：

```c
include/linux/init.h
 
#ifdef CONFIG_LTO_CLANG
  /*
   * With LTO, the compiler doesn't necessarily obey link order for
   * initcalls, and the initcall variable needs to be globally unique
   * to avoid naming collisions.  In order to preserve the correct
   * order, we add each variable into its own section and generate a
   * linker script (in scripts/link-vmlinux.sh) to ensure the order
   * remains correct.  We also add a __COUNTER__ prefix to the name,
   * so we can retain the order of initcalls within each compilation
   * unit, and __LINE__ to make the names more unique.
   */
  #define ___lto_initcall(c, l, fn, id, __sec) \
	static initcall_t __initcall_##c##_##l##_##fn##id __used \
		__attribute__((__section__( #__sec \
			__stringify(.init..##c##_##l##_##fn)))) = fn;
  #define __lto_initcall(c, l, fn, id, __sec) \
	___lto_initcall(c, l, fn, id, __sec)
 
  #define ___define_initcall(fn, id, __sec) \
	__lto_initcall(__COUNTER__, __LINE__, fn, id, __sec)
#else
  #define ___define_initcall(fn, id, __sec) \
	static initcall_t __initcall_##fn##id __used \
		__attribute__((__section__(#__sec ".init"))) = fn;
#endif
#endif
 
#define __define_initcall(fn, id) ___define_initcall(fn, id, .initcall##id)
```

下文会继续细化分析，这里提前提示：

\_\_define_initcall() 其实就是定义了一个 static initcall_t 的函数指针

```c
include/linux/init.h
 
typedef int (*initcall_t)(void);
typedef void (*exitcall_t)(void);
```

## 六、attribute详解

(1)、\_\_used

```c
include/linux/compiler_attributes.h

#define __used                          __attribute__((__used__))
```

这是一种 attribute 修饰属性的一种，意思是告诉编译器：这个静态符号在编译的时候，即使没有使用也要保留，不能优化掉。

(2)、\_\_attribute__ ((**section**(...)))
**attribute** 是 GNU C 的一大特色，可以用来修饰对象、函数、结构体类型等等。

这里用来修改 section，意思是将作用的函数放入指定的 section name 对应的段中。

(3)、 \_\_stringify()

```c
include/linux/stringify.h
 
#define __stringify_1(x...)	#x
#define __stringify(x...)	__stringify_1(x)
```

将 __stringify() 中内容字符串化。

## 七、例子

以静态的module_init定义为例：

假如，我们在驱动使用如下接口：

```c
module_init(hello_init);
```

那么，在编译的时候编译器会通过 initcall 接口产生：

```c
static initcall_t __initcall_1_23_hello_init6 __attribute__(__used) \
    __attribute__((__section__(".initcall6.init..1_23_hello_init"))) = hello_init;
```

### linux 编译后的initcall 函数

查看编译好的 **System.map：**

```bash
...
ffffffc012032ee0 d __initcall_223_42_trace_init_flags_sys_enterearly
ffffffc012032ee0 D __initcall_start
ffffffc012032ee0 D __setup_end
ffffffc012032ee8 d __initcall_224_66_trace_init_flags_sys_exitearly
ffffffc012032ef0 d __initcall_163_146_cpu_suspend_initearly
ffffffc012032ef8 d __initcall_151_267_asids_initearly
ffffffc012032f00 d __initcall_167_688_spawn_ksoftirqdearly
ffffffc012032f08 d __initcall_343_6656_migration_initearly
...
ffffffc012032f90 d __initcall_312_768_initialize_ptr_randomearly
ffffffc012032f98 D __initcall0_start
ffffffc012032f98 d __initcall_241_771_bpf_jit_charge_init0
ffffffc012032fa0 d __initcall_141_53_init_mmap_min_addr0
ffffffc012032fa8 d __initcall_209_6528_pci_realloc_setup_params0
ffffffc012032fb0 d __initcall_339_1143_net_ns_init0
ffffffc012032fb8 D __initcall1_start
ffffffc012032fb8 d __initcall_160_1437_fpsimd_init1
ffffffc012032fc0 d __initcall_181_669_tagged_addr_init1
...
ffffffc012033178 d __initcall_347_1788_init_default_flow_dissectors1
ffffffc012033180 d __initcall_360_2821_netlink_proto_init1
ffffffc012033188 D __initcall2_start
ffffffc012033188 d __initcall_165_139_debug_monitors_init2
ffffffc012033190 d __initcall_141_333_irq_sysfs_init2
...
ffffffc0120332b8 d __initcall_304_814_kobject_uevent_init2
ffffffc0120332c0 d __initcall_184_1686_msm_rpm_driver_init2s
ffffffc0120332c8 D __initcall3_start
ffffffc0120332c8 d __initcall_173_390_debug_traps_init3
ffffffc0120332d0 d __initcall_161_275_reserve_memblock_reserved_regions3
...
ffffffc012033370 d __initcall_132_5273_gsi_init3
ffffffc012033378 d __initcall_149_547_of_platform_default_populate_init3s
ffffffc012033380 D __initcall4_start
...
ffffffc012033878 D __initcall5_start
...
ffffffc0120339d8 d __initcall_317_1188_xsk_init5
ffffffc0120339e0 d __initcall_211_194_pci_apply_final_quirks5s
ffffffc0120339e8 d __initcall_168_680_populate_rootfsrootfs
ffffffc0120339e8 D __initcallrootfs_start
ffffffc0120339f0 D __initcall6_start
...
ffffffc012034b30 D __initcall7_start
...
ffffffc012034c88 d __initcall_150_554_of_platform_sync_state_init7s
ffffffc012034c90 d __initcall_123_29_alsa_sound_last_init7s
ffffffc012034c98 D __con_initcall_start
ffffffc012034c98 d __initcall_151_246_hvc_console_initcon
ffffffc012034c98 D __initcall_end
ffffffc012034ca0 D __con_initcall_end
```

从 System.map 得知：

* 从 __initcall_start ~ __initcall_end 所有函数指针都是连续的，相差8 个字节；
* \_\_initcall_start 就是第一个 early 级别的 initcall 函数指针，同理 __initcall0_start 就是第一个 level 0 级别的initcall 函数指针，以此类推；
* rootfs 级别的 initcall 函数是插在 level 5s 之后，level 6 级别之前；
* console 级别的函数在 level 7s 之后，__initcall_end 之前；
  当然通过命令 readelf 或者 objdump (objdump -h vmlinux.o)都能看到各个字段

```c
Sections:
Idx Name          Size      VMA               LMA               File off  Algn
  0 .initcall0.init 00000020  0000000000000000  0000000000000000  00000040  2**3
                  CONTENTS, ALLOC, LOAD, RELOC, DATA
  1 .initcall1.init 000001d0  0000000000000000  0000000000000000  00000060  2**3
                  CONTENTS, ALLOC, LOAD, RELOC, DATA
  2 .initcall2.init 00000138  0000000000000000  0000000000000000  00000230  2**3
                  CONTENTS, ALLOC, LOAD, RELOC, DATA
  3 .initcall2s.init 00000008  0000000000000000  0000000000000000  00000368  2**3
                  CONTENTS, ALLOC, LOAD, RELOC, DATA
  4 .initcall3.init 000000b0  0000000000000000  0000000000000000  00000370  2**3
                  CONTENTS, ALLOC, LOAD, RELOC, DATA
  5 .initcall3s.init 00000008  0000000000000000  0000000000000000  00000420  2**3
                  CONTENTS, ALLOC, LOAD, RELOC, DATA
  6 .initcall4.init 000004f0  0000000000000000  0000000000000000  00000428  2**3
                  CONTENTS, ALLOC, LOAD, RELOC, DATA
  7 .initcall4s.init 00000008  0000000000000000  0000000000000000  00000918  2**3
                  CONTENTS, ALLOC, LOAD, RELOC, DATA
  8 .initcall5.init 00000168  0000000000000000  0000000000000000  00000920  2**3
                  CONTENTS, ALLOC, LOAD, RELOC, DATA
  9 .initcall5s.init 00000008  0000000000000000  0000000000000000  00000a88  2**3
                  CONTENTS, ALLOC, LOAD, RELOC, DATA
 10 .initcall6.init 00001140  0000000000000000  0000000000000000  00000a90  2**3
                  CONTENTS, ALLOC, LOAD, RELOC, DATA
 11 .initcall7.init 00000140  0000000000000000  0000000000000000  00001bd0  2**3
                  CONTENTS, ALLOC, LOAD, RELOC, DATA
 12 .initcall7s.init 00000028  0000000000000000  0000000000000000  00001d10  2**3
                  CONTENTS, ALLOC, LOAD, RELOC, DATA
 13 .con_initcall.init 00000008  0000000000000000  0000000000000000  00001d38  2**3
                  CONTENTS, ALLOC, LOAD, RELOC, DATA
 14 .initcallearly.init 000000b8  0000000000000000  0000000000000000  00001d40  2**3
                  CONTENTS, ALLOC, LOAD, RELOC, DATA
 15 .initcallrootfs.init 00000008  0000000000000000  0000000000000000  00001df8  2**3
                  CONTENTS, ALLOC, LOAD, RELOC, DATA
```