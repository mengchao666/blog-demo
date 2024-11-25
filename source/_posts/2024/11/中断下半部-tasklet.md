---
title: 中断下半部-tasklet
tags: [kernel]
categories: [kernel]
permalink: 
poster:
  topic: null
  headline: null
  caption: null
  color: null
date: 2024-11-25 23:02:32
topic: linux_kernel
description:
cover:
banner:
references:
---
tasklet 是中断下半部的一种实现机制，主要用于小任务处理，耗时较短、不能阻塞的任务，用tasklet处理较合适。对于耗时较长，可以用work queue（工作队列）来处理。

tasklet和内核定时器timer_list都是通过软中断方式来实现的。

### 一、tasklet结构体

中断下半部用结构体tasklet_struct来表示

```c
#include <linux/interrupt.h>

struct tasklet_struct
{
    struct tasklet_struct *next;
    unsigned long state;
    atomic_t count;
    void (*func)(unsigned long);
    unsigned long data;
};
```

其中，state有2位：<br />1）bit0：表示TASKLET_STATE_SCHED<br />等于1，表示已经执行了tasklet_schedule，该把tasklet放入队列了。tasklet_schedule会判断该位，如果已经等于1，那么它就不会再次把tasklet放入队列。

2）bit1：表示TASKLET_STATE_RUN<br />等于1，表示正在运行tasklet中的func函数。函数执行完毕后，内核会把该位清0。

count表示该tasklet是否使能：值0表示使能了，非0表示被禁止了。对于count非0的tasklet，func()不会被执行。

data 是传递给func()的参数。

### 二、初始化tasklet_strcut

#### 静态初始化：

宏初始化tasklet_struct

```c
#define DECLARE_TASKLET(name, func, data) \
struct tasklet_struct name = { NULL, 0, ATOMIC_INIT(0), func, data };

#define DECLARE_TASKLET_DISABLED(name, func, data) \
struct tasklet_struct name = { NULL, 0, ATOMIC_INIT(1), func, data };
```

使用DECLARE_TASKLET定义的tasklet结构体，它是使能的。<br />使用DECLARE_TASKLET_DISABLED定义的tasklet结构体，它是禁止的。使用之前要先调用tasklet_enable使能之。

##### 动态初始化：

也可用函数tasklet_init()初始化tasklet结构体：
data是func的参数

```c
tasklet_init(struct tasklet_struct *t,
             void (*func)(unsigned long), unsigned long data);
```

### 三、使能/禁止tasklet

tasklet_enable将count加1；tasklet_disable将count减1。

```c
static inline void tasklet_enable(struct tasklet_struct *t);  // 使能

static inline void tasklet_disable(struct tasklet_struct *t); // 禁止
```

### 四、调度tasklet

将tasklet放入链表，并设置它的TASKLET_STATE_SCHED状态为1。

```c
static inline void tasklet_schedule(struct tasklet_struct *t);
```

### 五、kill tasklet

从链表中删除tasklet。如果一个tasklet未被调度，tasklet_kill会将它的TASKLET_STATE_SCHED状态清0；如果一个tasklet已被调度，tasklet_kill会等待它执行完毕，再把它的TASKLET_STATE_SCHED状态清0。<br />通常，在卸载驱动程序（module_exit）时，调用task_kill。

tasklet_kill_immediate 与tasklet_kill区别是，前者会立即移除tasklet，二不论tasklet是否处于TASKLET_STATE_SCHED状态。

```c
void tasklet_kill(struct tasklet_struct *t); // 移除tasklet

void tasklet_kill_immediate(struct tasklet_struct *t, unsigned int cpu); // 立即移除tasklet
```

### 六、tasklet使用方法

先定义tasklet实例，需要使用时调用tasklet_schedule，驱动卸载前调用tasklet_kill。tasklet_schedule只是将tasklet放入内核队列，其func函数会在软中断执行过程中被调用。

### 七、tasklet内部实现机制

前面讲过，tasklet是通过软中断实现，属于TASKLET_SOFTIRQ类型的软中断。入口函数tasklet_action。

```c
// kernel/softirq.c

void __init softirq_init(void)
{
    int cpu;

    for_each_possible_cpu(cpu) {
        per_cpu(tasklet_vec, cpu).tail =
            &per_cpu(tasklet_vec, cpu).head;
        per_cpu(tasklet_hi_vec, cpu).tail =
            &per_cpu(tasklet_hi_vec, cpu).head;
    }

    open_softirq(TASKLET_SOFTIRQ, tasklet_action); // 注册TASKLET_SOFTIRQ类型软中断(普通软中断)及其处理函数
    open_softirq(HI_SOFTIRQ, tasklet_hi_action);   // 高优先级软中断
}
```

在软中断的初始化（softirq_init）末尾，通过open_softirq注册TASKLET_SOFTIRQ类型的软中断（即tasklet）及其处理函数（即tasklet_action）。

驱动程序调用tasklet_schedule时，会设置tasket的state为TASKLET_STATE_SCHED，并把它放入某个链表：

```c
static inline void tasklet_schedule(struct tasklet_struct *t)
{
    // 1. 如果未设置为SCHED，则设置为SCHED并放入队列
    if (!test_and_set_bit(TASKLET_STATE_SCHED, &t->state)) // 设置tasklet.state为TASKLET_STATE_SCHED
        __tasklet_schedule(t);
}

void __tasklet_schedule(struct tasklet_struct *t)
{
    unsigned long flags;

    local_irq_save(flags); // 保存中断
    // 2. 放入队列
    t->next = NULL;
    *__this_cpu_read(tasklet_vec.tail) = t;
    __this_cpu_write(tasklet_vec.tail, &(t->next));
    raise_softirq_irqoff(TASKLET_SOFTIRQ); // 唤醒中断, 会导致调用该类型软中断对应的处理函数
    local_irq_restore(flags); // 恢复中断到flags状态
}
EXPORT_SYMBOL(__tasklet_schedule);
```

产生硬件中断时，讹你好处理完硬件中断后，会处理软中断。对于TASKLET_SOFTIRQ软中断，会调用tasklet_action函数。

执行过程是：从队列中找到tasklet，进行状态判断后执行func函数，从队列中删除tasklet。<br />可知：<br />1）tasklet_schedule 调度tasklet时，其中的函数并不会立即执行，而只是把tasklet放入队列；<br />2）调用一次tasklet_schedule，只会导致tasklet的函数被执行一次；<br />3）如果tasklet的函数尚未执行，多次调用tasklet_schedule也是少的，只会放入队列一次。

普通软中断处理函数tasklet_action：

```c
static __latent_entropy void tasklet_action(struct softirq_action *a)
{
    struct tasklet_struct *list;

    local_irq_disable();
    list = __this_cpu_read(tasklet_vec.head);
    __this_cpu_write(tasklet_vec.head, NULL);
    __this_cpu_write(tasklet_vec.tail, this_cpu_ptr(&tasklet_vec.head));
    local_irq_enable();

    while (list) {
        struct tasklet_struct *t = list;

        list = list->next; // 1. 从列表中去除每一项

        if (tasklet_trylock(t)) {
            if (!atomic_read(&t->count)) {
                // 2. 判断：如果不是SCHED状态，就是有BUG
                if (!test_and_clear_bit(TASKLET_STATE_SCHED,
                            &t->state))
                    BUG();
                t->func(t->data); // 3. 执行tasklet的func
                tasklet_unlock(t);
                continue;
            }
            tasklet_unlock(t);
        }

        local_irq_disable();
        // 4. 从队列中取出
        t->next = NULL;
        *__this_cpu_read(tasklet_vec.tail) = t;
        __this_cpu_write(tasklet_vec.tail, &(t->next));

        __raise_softirq_irqoff(TASKLET_SOFTIRQ);
        local_irq_enable();
    }
}
```