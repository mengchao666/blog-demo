---
title: 中断下半部-workqueue
tags: [kernel]
categories: [kernel]
permalink: 
poster:
  topic: null
  headline: null
  caption: null
  color: null
date: 2024-11-25 23:03:33
topic: linux_kernel
description:
cover:
banner:
references:
---
工作队列（work queue）是中断下半部的一种实现机制，主要用于耗时任务处理，由内核线程代表进程执行。工作队列运行于进程上下文，因此允许阻塞。

运行工作队列的内核线程，称为工作者线程（worker thread），可以使用系统默认的，也可以自行创建（通常无必要理由不推荐）。

使用工作队列方式：1）初始化工作队列；2）将“工作”（work）放入“工作队列中”。这样，对应的内核线程就会取出“工作”，执行其中的函数。

工作队列缺点：多个工作挤在某个内核线程中依次序执行，前面的函数如果执行得很慢，就会影响到后面的函数。

### 一、内核数据结构与函数

work queue有关数据结构和函数，都位于<linux/workqueue.h>。

### 二、work_struct结构体

一个work_struct实例代表一个“工作”，工作包含了用户想要要执行的任务。

work_struct结构体定义：

```c
struct work_struct {
    atomic_long_t data;
    struct list_head entry;
    work_func_t func;          // 处理函数
#ifdef CONFIG_LOCKDEP
    struct lockdep_map lockdep_map;
#endif
};

typedef void (*work_func_t)(struct work_struct *work);
```

使用work queue时，步骤如下：<br />1）构造一个work_struct实例，设置处理函数。<br />2）把work_struct放入工作队列，内核线程会运行work中的函数（func）。

### 三、使用work queue

### 创建work

##### 静态创建

宏DECLARE_WORK用来定义一个work_struct结构体，需要指定它的处理函数。<br />宏DECLARE_DELAYED_WORK用来定义一个delayed_work结构体，也需要指定它的处理函数。“delayed”指延时，意思是要让该“工作”运行时，可以通过该宏指定延时的时间。

```c
#define DECLARE_WORK(n, f)                        \
    struct work_struct n = __WORK_INITIALIZER(n, f)

#define DECLARE_DELAYED_WORK(n, f)                    \
    struct delayed_work n = __DELAYED_WORK_INITIALIZER(n, f, 0)
```

delayed_work结构体，其实是一个work_struct和一个timer_list等成员的复合结构。

```c
struct delayed_work {
    struct work_struct work; // 工作队列的工作
    struct timer_list timer; // 超时时间

    /* target workqueue and CPU ->timer uses to queue ->work */
    struct workqueue_struct *wq;
    int cpu;
};
```

##### 动态创建

宏INIT_WORK用来初始化work_struct结构体：

```c
#define INIT_WORK(_work, _func)                        \
    __INIT_WORK((_work), (_func), 0)
```

### 四、创建工作队列

Linux系统中已有现成的system_wq等工作队列，使用工作队列时，通常推荐用现成的。

```c
// Linux中现成的工作队列

extern struct workqueue_struct *system_wq;
extern struct workqueue_struct *system_highpri_wq;
extern struct workqueue_struct *system_long_wq;
extern struct workqueue_struct *system_unbound_wq;
extern struct workqueue_struct *system_freezable_wq;
extern struct workqueue_struct *system_power_efficient_wq;
extern struct workqueue_struct *system_freezable_power_efficient_wq;
```

如果需要自行创建，也有办法，可以使用create_workqueue或create_singlethread_workqueue。<br />create_workqueue会在SMP系统中，针对每个CPU，都创建一个内核线程和创建的工作队列对应。<br />create_singlethread_workqueue 只会有一个内核线程与工作队列对应。

### 五、销毁工作队列

与创建工作队列相对的，是销毁工作队列，可以调用destroy_workqueue来执行该操作。

```c
void destroy_workqueue(struct workqueue_struct *wq);
```

### 六、调度执行work

schedule_work调度执行一个具体的work，执行的work将会被挂入Linux提供的（默认system_wq）工作队列。

```c
static inline bool schedule_work(struct work_struct *work);
```

如果想延迟执行work，可以调用schedule_delayed_work ，其功能类似于schedule_work，不过多了一个延迟。

```c
static inline bool schedule_delayed_work(struct delayed_work *dwork,
                     unsigned long delay);
```

queue_work 跟schedule_work类似，区别在于schedule_work是在系统默认的工作队列上执行一个work，而queue_work 需要自行指定工作队列。

其实，schedule_work是利用queue_work实现的，例如系统默认的工作队列system_wq：

```c
static inline bool schedule_work(struct work_struct *work)
{
    return queue_work(system_wq, work);
}
```

queue_delayed_work 跟schedule_delayed_work 类似，区别在于schedule_delayed_work 是在系统默认的工作队列上执行一个work，queue_delayed_work需要自行指定工作队列。类似地，schedule_delayed_work也是依赖于queue_delayed_work实现的。

```c
static inline bool schedule_delayed_work(struct delayed_work *dwork,
                     unsigned long delay)
{
    return queue_delayed_work(system_wq, dwork, delay);
}
```

### 七、等待work

flush_work 等待一个work执行完毕。如果该work已经被放入队列，那么本函数等它执行完毕，并且返回true；如果该work已经执行完毕才调用本函数，那么直接返回false。

```c
bool flush_work(struct work_struct *work);
```

flush_delayed_work 等待一个delayed_work执行完毕。如果这个delayed_work已经被放入队列，那么本函数等它执行完毕，并且返回true；如果这个delayed_work已经执行完毕才调用本函数，那么直接返回false。

```c
bool flush_delayed_work(struct delayed_work *dwork);
```

TIPS：前面提到过，delayed_work是一个复合了work_struct，timer_list等成员的结构体。

### 八、等待work queue

flush_work是等待一个work执行完毕，而flush_workqueue是等待一个工作队列上所有work执行完毕。

```c
void flush_workqueue(struct workqueue_struct *wq)
```

---

### 九、work queue的内部机制

Linux内核2.x 版本中，创建workqueue时会同步创建内核线程；<br />Linux内核4.x 版本中，内核线程和workqueue分开创建，较为复杂。

### Linux 2.x的工作队列创建过程

kernel/workqueue：

```c
init_workqueues
keventd_wq = create_workqueue("events"); // 创建名为"events"的工作队列
    __create_workqueue((name), 0, 0)
        for_each_possible_cpu(cpu) {
            err = create_workqueue_thread(cwq, cpu); // 创建用于工作队列的内核线程
                    p = kthread_create(worker_thread, cwq, fmt, wq->name, cpu); // 创建内核线程
```

对于每个CPU，都创建一个名为“events/n”的内核线程，n是处理器编号，从0开始。

创建workqueue的同时，创建内核线程。

每个CPU上都有一个cpu_workqueue_struct，而每个cpu_workqueue_struct下只有1个线程用于work queue执行work。所有内核线程可以从同一个work queue取work。

### Linux 4.x的工作队列创建过程

```c
init_workqueues
/* initialize CPU pools */
for_each_possible_cpu(cpu) {
    for_each_cpu_worker_pool(pool, cpu) {
         /* 对每一个CPU都创建2个worker_pool结构体，它是含有ID的 */
         /*  一个worker_pool对应普通优先级的work，第2个对应高优先级的work */
}

/* create the initial worker */
for_each_online_cpu(cpu) {
    for_each_cpu_worker_pool(pool, cpu) {
        /* 对每一个CPU的每一个worker_pool，创建一个worker */
/* 每一个worker对应一个内核线程 */
        BUG_ON(!create_worker(pool));
    }
}
```

create_worker：

```c
static struct worker *create_worker(struct worker_pool *pool)
{
    struct worker *worker = NULL;
    int id = -1;
    char id_buf[16];

    /* ID is needed to determine kthread name */
    id = ida_simple_get(&pool->worker_ida, 0, 0, GFP_KERNEL);
    if (id < 0)
        goto fail;

    worker = alloc_worker(pool->node);
    if (!worker)
        goto fail;

    worker->pool = pool;
    worker->id = id;
  
    if (pool->cpu >= 0)
        snprintf(id_buf, sizeof(id_buf), "%d:%d%s", pool->cpu, // 在哪个CPU上运行
             id, // poll中第几个线程
             pool->attrs->nice < 0  ? "H" : "");  // H: 高优先级
    else
        snprintf(id_buf, sizeof(id_buf), "u%d:%d", pool->id, id);

    worker->task = kthread_create_on_node(worker_thread, worker, pool->node,
                          "kworker/%s", id_buf); // 内核线程的名字
    ...
}
```

创建号内核线程（"kworker/n:id"）后，再创建workqueue

```c
init_workqueues
    system_wq = alloc_workqueue("events", 0, 0);
        __alloc_workqueue_key
            wq = kzalloc(sizeof(*wq) + tbl_size, GFP_KERNEL); // 分配workqueue_struct
            alloc_and_link_pwqs(wq)  // 跟worker_poll建立联系
```

每个CPU对应2个woker_pool：一个普通的worker_pool，一个高优先级的worker_pool。每个线程池包含多个内核线程，用于执行同一个worker queue的work。work_pool的线程名，形如"kworker/n:idH"，n代表CPU编号，id是子线程编号，H代表高优先级，如果普通优先级则为空。

对于CPU 0：<br />普通worker_pool线程名，形如"kworker/0:0"，"kworker/0:1"，"kworker/0:2"。:<br />高优先级的worker_pool线程名，形如"kworker/0:0H"，"kworker/0:1H"，"kworker/0:2H"。