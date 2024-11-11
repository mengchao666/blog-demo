---
title: addr2line
tags: [Linux工具]
categories: [Linux工具]
permalink: 
poster:
  topic: null
  headline: null
  caption: null
  color: null
date: 2024-11-10 22:39:55
topic: linux_tool
description:
cover:
banner:
references:
---

addr2line是一个将地址转换为文件名和行号的工具。给定可执行文件(如exe/a.out等)中的地址或可重定位对象(如so,ko等)部分中的偏移量，它会使用调试信息来确定与之相关的文件名和行数。

用户态coredump，一般使用gdb调试接口，gdb一般封装了addr2line，可以解析文件名和行号。如果环境配置了不生成coredump，可以使用addr2line调试。`所以addr2line一般解析内核Call trace使用较多。`

## 一、使用方法

```c
基本用法：
addr2line [选项] [地址]
```

常用选项如下：

| 选项      | 描述                          |
| ------- | ------- |
| -e       | 设置输入文件名称，默认为a.out |
| -i       | 解析内联函数                  |
| -f      | 显示函数名                    |
| -C       | 解析函数名                    |
| -p       | 以好读的方式显示              |

需要注意的是使用addr2line的时候，可执行文件或重定位文件一定是要带调试信息的

## 二、用户态普通程序崩溃

使用方法

```c
addr2line -e 进程名 IP指令地址 -f
```

用户态程序崩溃，当没有coredump产生时，可以使用如下方法
假设我们的程序名称为segfault,当程序崩溃是，dmesg日志中会有报错信息:

```sh
[root@localhost ~]# dmesg
[134563.793925] segfault[53791]: segfault at 0 ip 0000000000400546 sp 00007fff7956af70 error 6 in segfault[400000+1000]
[134563.793946] Code: 01 5d c3 90 c3 66 66 2e 0f 1f 84 00 00 00 00 00 0f 1f 40 00 f3 0f 1e fa eb 8a 55 48 89 e5 48 c7 45 f8 00 00 00 00 48 8b 45 f8 <c7> 00 00 00 00 00 b8 00 00 00 00 5d c3 66 2e 0f 1f 84 00 00 00 00
```

此时我们注意到IP指令地址为0000000000400546
使用addr2line查看程序挂的位置：

```c
addr2line -e segfault 0x0000000000400546 -f
```

## 三、动态链接库程序崩溃

使用方法

```c
addr2line -e 动态链接库名称 IP指令地址-基地址 -f
```

假设我们有一个程序名为test，链接了一个libfoo.so，程序运行时崩溃，dmesg查看日志如下：

```c
[root@localhost ~]# dmesg
[70567.416655] test[27722]: segfault at 0 ip 00007ffa1f588580 sp 00007fffa964e698 error 6 in libfoo.so[7ffa1f588000+1000]
[70567.427374] Code: ff e8 64 ff ff ff c6 05 bd 0a 20 00 01 5d c3 0f 1f 00 c3 0f 1f 80 00 00 00 00 f3 0f 1e fa e9 77 ff ff ff 0f 1f 80 00 00 00 00 <c7> 04 25 00 00 00 00 00 00 00 00 0f 0b 00 00 00 f3 0f 1e fa 48 83
```

根据日志可知，段错误发生的位置是在test进程调用的`libfoo.so`库里，我们先使用ldd找到动态库的位置，如下：

```c
[root@localhost 69]# ldd test
        linux-vdso.so.1 (0x00007ffd15b24000)
        libfoo.so => ./libfoo.so (0x00007f8c01879000)
        libc.so.6 => /lib64/libc.so.6 (0x00007f8c014b4000)
        /lib64/ld-linux-x86-64.so.2 (0x00007f8c01a7b000)
```

00007ffa1f588580为程序崩溃点IP指令地址，使用dmesg消息中ip指令地址减去动态库基址值(00007ffa1f588580 -7ffa1f588000=580), 差值0x580为错误点在动态库的地址，调用addr2line, 注意-e参数后文件名改为动态库名:

```c
addr2line -e libfoo.so 580 -f -p
```

## 四、内核驱动程序运行崩溃

使用方法：

```c
addr2line -e xxx.ko 地址偏移量 -f
```

本人所用主机即属于一旦发生Oops，就会触发panic，因此总是无法查看Oops时的dmesg日志，经查阅资料，发现是内核参数panic_on_oops的原因导致的，因为该参数被设置为1，所以Oops会触发panic，从而导致机器总是死机重启，无法查看Oops时的dmesg日志。下面提供两种方法修改Oops内核参数，使其不会在Oops的时候触发panic导致死机重启。

方法一：修改 /proc下内核参数文件内容，临时生效，重启后失效。

```c
echo 0 > /proc/sys/kernel/panic_on_oops
```

方法二：修改/etc/sysctl.conf 文件的内核参数来永久更改。

```shell
[root@localhost ~]# vi /etc/sysctl.conf
[root@localhost ~]# cat /etc/sysctl.conf
# sysctl settings are defined through files in
# /usr/lib/sysctl.d/, /run/sysctl.d/, and /etc/sysctl.d/.
#
# Vendors settings live in /usr/lib/sysctl.d/.
# To override a whole file, create a new file with the same in
# /etc/sysctl.d/ and put new settings there. To override
# only specific settings, add a file with a lexically later
# name in /etc/sysctl.d/ and put new settings there.
#
# For more information, see sysctl.conf(5) and sysctl.d(5).
kernel.panic_on_oops=0
[root@localhost ~]# cat /proc/sys/kernel/panic_on_oops
1 
[root@localhost ~]# sysctl -p
kernel.panic_on_oops = 0
[root@localhost ~]#
[root@localhost ~]# cat /proc/sys/kernel/panic_on_oops
0
```

假设内核某ko运行后发生如下错误：

```c
[root@localhost ~]# dmesg
[ 1039.918606] my_oops_init
[ 1039.918616] BUG: unable to handle kernel NULL pointer dereference at 0000000000000000
[ 1039.926442] PGD 0 P4D 0
[ 1039.928979] Oops: 0002 [#1] SMP NOPTI
[ 1039.932637] CPU: 34 PID: 3843 Comm: insmod Kdump: loaded Tainted: G           OE    --------- -  - 4.18.0-394.el8.x86_64 #1
[ 1039.943756] Hardware name: New H3C Technologies Co., Ltd. H3C UniServer R4950 G5/RS45M2C9SB, BIOS 5.37 09/30/2021
[ 1039.954000] RIP: 0010:do_oops+0x5/0x11 [oops]
[ 1039.958364] Code: Unable to access opcode bytes at RIP 0xffffffffc02e6fdb.
[ 1039.965231] RSP: 0018:ffffb9d40a8c7cb0 EFLAGS: 00010246
[ 1039.970449] RAX: 000000000000000c RBX: 0000000000000000 RCX: 0000000000000000
[ 1039.977573] RDX: 0000000000000000 RSI: ffff98942ee96758 RDI: ffff98942ee96758
[ 1039.984697] RBP: ffffffffc02e7011 R08: 0000000000000000 R09: c0000000ffff7fff
[ 1039.991822] R10: 0000000000000001 R11: ffffb9d40a8c7ad8 R12: ffffffffc02e9000
[ 1039.998944] R13: ffffffffc02e9018 R14: ffffffffc02e91d0 R15: 0000000000000000
[ 1040.006069] FS:  00007f1b8d93b740(0000) GS:ffff98942ee80000(0000) knlGS:0000000000000000
[ 1040.014145] CS:  0010 DS: 0000 ES: 0000 CR0: 0000000080050033
[ 1040.019884] CR2: ffffffffc02e6fdb CR3: 0000000145c02000 CR4: 0000000000350ee0
[ 1040.027008] Call Trace:
[ 1040.029454]  my_oops_init+0x16/0x19 [oops]
[ 1040.033550]  do_one_initcall+0x46/0x1d0
[ 1040.037390]  ? do_init_module+0x22/0x220
[ 1040.041318]  ? kmem_cache_alloc_trace+0x142/0x280
[ 1040.046023]  do_init_module+0x5a/0x220
[ 1040.049777]  load_module+0x14ba/0x17f0
[ 1040.053530]  ? __do_sys_finit_module+0xb1/0x110
[ 1040.058059]  __do_sys_finit_module+0xb1/0x110
[ 1040.062411]  do_syscall_64+0x5b/0x1a0
[ 1040.066077]  entry_SYSCALL_64_after_hwframe+0x65/0xca
[ 1040.071130] RIP: 0033:0x7f1b8c8509bd
[ 1040.074701] Code: ff c3 66 2e 0f 1f 84 00 00 00 00 00 90 f3 0f 1e fa 48 89 f8 48 89 f7 48 89 d6 48 89 ca 4d 89 c2 4d 89 c8 4c 8b 4c 24 08 0f 05 <48> 3d 01 f0 ff ff 73 01 c3 48 8b 0d 9b 54 38 00 f7 d8 64 89 01 48
[ 1040.093446] RSP: 002b:00007ffc4df0a968 EFLAGS: 00000246 ORIG_RAX: 0000000000000139
[ 1040.101004] RAX: ffffffffffffffda RBX: 00005653fb1997d0 RCX: 00007f1b8c8509bd
[ 1040.108126] RDX: 0000000000000000 RSI: 00005653f980c8b6 RDI: 0000000000000003
[ 1040.115251] RBP: 00005653f980c8b6 R08: 0000000000000000 R09: 00007f1b8cbd9760
[ 1040.122375] R10: 0000000000000003 R11: 0000000000000246 R12: 0000000000000000
[ 1040.129498] R13: 00005653fb1997b0 R14: 0000000000000000 R15: 0000000000000000
[ 1040.136623] Modules linked in: oops(OE+) binfmt_misc xt_CHECKSUM ipt_MASQUERADE xt_conntrack ipt_REJECT nf_reject_ipv4 nft_compat nft_counter nft_chain_nat nf_nat nf_conntrack nf_defrag_ipv6 nf_defrag_ipv4 nf_tables nfnetlink rpcsec_gss_krb5 auth_rpcgss nfsv4 dns_resolver nfs lockd grace fscache bridge stp llc intel_rapl_msr intel_rapl_common amd64_edac_mod edac_mce_amd amd_energy kvm_amd kvm irqbypass ipmi_ssif pcspkr crct10dif_pclmul crc32_pclmul ghash_clmulni_intel rapl joydev ccp sp5100_tco i2c_piix4 k10temp ptdma acpi_ipmi ipmi_si sunrpc vfat fat xfs libcrc32c sd_mod t10_pi sg crc32c_intel ast drm_vram_helper drm_kms_helper syscopyarea sysfillrect sysimgblt fb_sys_fops drm_ttm_helper ttm ahci drm libahci nfp(OE) igb libata dca i2c_algo_bit dm_mirror dm_region_hash dm_log dm_mod ipmi_devintf ipmi_msghandler
[ 1040.208357] CR2: 0000000000000000
[ 1040.211668] ---[ end trace b69c1e8998070273 ]---
[ 1040.230185] RIP: 0010:do_oops+0x5/0x11 [oops]
[ 1040.234540] Code: Unable to access opcode bytes at RIP 0xffffffffc02e6fdb.
[ 1040.241409] RSP: 0018:ffffb9d40a8c7cb0 EFLAGS: 00010246
[ 1040.246626] RAX: 000000000000000c RBX: 0000000000000000 RCX: 0000000000000000
[ 1040.253750] RDX: 0000000000000000 RSI: ffff98942ee96758 RDI: ffff98942ee96758
[ 1040.260876] RBP: ffffffffc02e7011 R08: 0000000000000000 R09: c0000000ffff7fff
[ 1040.267998] R10: 0000000000000001 R11: ffffb9d40a8c7ad8 R12: ffffffffc02e9000
[ 1040.275124] R13: ffffffffc02e9018 R14: ffffffffc02e91d0 R15: 0000000000000000
[ 1040.282247] FS:  00007f1b8d93b740(0000) GS:ffff98942ee80000(0000) knlGS:0000000000000000
[ 1040.290323] CS:  0010 DS: 0000 ES: 0000 CR0: 0000000080050033
[ 1040.296061] CR2: ffffffffc02e6fdb CR3: 0000000145c02000 CR4: 0000000000350ee0
```

Oops: 0002 -- 错误码
Oops: [#1] -- Oops发生的次数
CPU: 34 -- 表示Oops是发生在CPU34上
关键信息如下，这里提示在操作函数do_oops的时候出现异常，地址偏移量0x5：

```sh
[ 1039.954000] RIP: 0010:do_oops+0x5/0x11 [oops]
```

为什么这条信息关键? 因为其含有指令指针RIP；指令指针IP/EIP/RIP的基本功能是指向要执行的下一条地址。在8080 8位微处理器上的寄存器名称是PC（program counter，程序计数器），从8086起，被称为IP（instruction pointer，指令指针）。主要区别在与PC指向正在执行的指令，而IP指向下一条指令。在64位模式下，指令指针是RIP寄存器。这个寄存器保存着下一条要执行的指令的64位地址偏移量。64位模式支持一种新的寻址模式，被称为RIP相对寻址。使用这个模式，有效地址的计算方式变为RIP（指向下一条指令）加上位移量。

由此可以看出内核执行到do_oops+0x5/0x11这个地址的时候出现异常，我们只需要找到这个地址对应的代码即可。

do_oops指示了是在do_oops函数中出现的异常， 0x5表示出错的地址偏移量， 0x11表示do_oops函数的大小。

```c
打印格式：do_oops+0x5/0x11 [oops] 
即：symbol+offset/size [module] 
symbol: 符号 offset：地址偏移量 size：函数的长度 module: 所属内核模块
```

使用如下命令解析

```c
addr2line -e oops.ko 0x5 -f -p
```
