---
title: 内核定时器&hrtime高精度定时器
tags: [kernel]
categories: [kernel]
permalink: 
poster:
  topic: null
  headline: null
  caption: null
  color: null
date: 2024-11-25 23:14:19
topic: linux_kernel
description:
cover:
banner:
references:
---
### 一、定时器初始化

`void hrtimer_init(struct hrtimer *timer, clockid_t clock_id, enum hrtimer_mode mode)；`

```c
 参数timer是hrtimer指针，
 参数clock_id有如下常用几种选项：
        CLOCK_REALTIME	//实时时间，如果系统时间变了，定时器也会变
        CLOCK_MONOTONIC	//递增时间，不受系统影响
 参数mode有如下几种选项：
 	HRTIMER_MODE_ABS = 0x0,		/* 绝对模式 */
	HRTIMER_MODE_REL = 0x1,		/* 相对模式 */
	HRTIMER_MODE_PINNED = 0x02,	/* 和CPU绑定 */
	HRTIMER_MODE_ABS_PINNED = 0x02, /* 第一种和第三种的结合 */
	HRTIMER_MODE_REL_PINNED = 0x03, /* 第二种和第三种的结合 */
```

### 二、启动定时器

`hrtimer_start(struct hrtimer *timer, ktime_t tim, const enum hrtimer_mode mode)；`

```c
参数timer是hrtimer指针
参数tim是时间，可以使用ktime_set()函数设置时间，
参数mode和初始化的mode参数一致
```

### 三、设置超时时间

```c
/*
 * 单位为秒和纳秒组合
 */
ktime_t ktime_set(const long secs, const unsigned long nsecs)；
 
/* 设置超时时间，当定时器超时后可以用该函数设置下一次超时时间 */
hrtimer_forward_now(struct hrtimer *timer, ktime_t interval)
```

### 四、设置回调函数

```c
struct hrtimer timer;
timer.function = hr_callback;
```

### 五、关闭定时器

`int hrtimer_cancel(struct hrtimer *timer)；`

使用例子：hrtimer.c

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/hrtimer.h>
#include <linux/jiffies.h>
 
//定义一个hrtimer
static struct hrtimer timer;
ktime_t kt;
 
//定时器回调函数
static enum hrtimer_restart hrtimer_hander(struct hrtimer *timer)
{
    printk("I am in hrtimer hander\r\n");
    hrtimer_forward(timer,timer->base->get_time(),kt);//hrtimer_forward(timer, now, tick_period);
    return HRTIMER_RESTART;  //重启定时器
}
 
static int __init test_init(void)
{
    printk("---------%s-----------\r\n",__func__);
 
    kt = ktime_set(0,1000000);// 0s  1000000ns  = 1ms 定时
    hrtimer_init(&timer,CLOCK_MONOTONIC,HRTIMER_MODE_REL);
    hrtimer_start(&timer,kt,HRTIMER_MODE_REL);
    timer.function = hrtimer_hander; //此处设置了定时器到时间后的回调函数
    return 0;
}
 
static void __exit test_exit(void)
{
    hrtimer_cancel(&timer);
    printk("------------test over---------------\r\n");
}
 
module_init(test_init);
module_exit(test_exit);
MODULE_LICENSE("GPL");
```