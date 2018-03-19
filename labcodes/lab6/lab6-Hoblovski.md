# 操作系统 lab6

## 使用 Round Robin 调度算法 (不需要编码)

### 请理解并分析 `sched_calss` 中各个函数指针的用法, 并接合 Round Robin 调度算法描 ucore 的调度执行过程
```C
struct sched_class {
    // ...
    const char *name;

    // 初始化调度队列
    void (*init)(struct run_queue *rq);

    // 有新的进程进入 RUNNABLE 态, 将其加入调度队列中
    void (*enqueue)(struct run_queue *rq, struct proc_struct *proc);

    // 调度队列中某一个进程从 RUNNABLE 变成 RUNNING, 其出队
    void (*dequeue)(struct run_queue *rq, struct proc_struct *proc);

    // 从调度队列中选择一个 RUNNABLE 的进程, 其应该下一次被调度执行
    struct proc_struct *(*pick_next)(struct run_queue *rq);

    // 每次时钟中断时刷新调度队列信息
    void (*proc_tick)(struct run_queue *rq, struct proc_struct *proc);
};
```

ucore 的调度过程
1. 在 `init_sched()` 中, 初始化调度器

2. 每当进程被唤醒 (即 SLEEPING -> RUNNABLE, 调用`wakeup_proc()`) 时, 都会被加入调度队列.
同时如果进程是因为用尽时间片而被迫终止 (即 RUNNING -> RUNNABLE) 时, 也会被加入调度队列.

3. 每次执行 `schedule()` 时, 其通过 `pick_next()` 选择一个下一时间片运行的进程, 并让其出队 (RUNNABLE -> RUNNING)

4. 导致 `schedule()` 的事件有
    * 当前进程调用 yield (或 exit, wait) 放弃 CPU
    * 时钟中断处理迫使当前程序放弃 CPU
    * CPU 空闲时循环查找可调度的进程

### 请在实验报告中简要说明如何设计实现”多级反馈队列调度算法“, 给出概要设计, 鼓励给出详细设计

* 在调度器的实现中, 需要有多个调度队列表示不同的进程级别 (i.e. 优先级), 每个都是一个 `sched_class`, 有完整的出入队和时钟处理

* 调度器每次按照一定的算法选择一个队列 (e.g. 对于队列间是 RR 算法, 则维护一个在各个队列间轮转的指针即可)
之后再从那个队列中出队 / 入队

* 提供反馈机制: 在进程出队时检查其是否用完时间片 (`proc->time_slice == 0`), 如果用完则降级 (或者用完若干次后降一级).
进程的级别是它的一个基本属性, 在其 `struct proc.priority` 中

简要的说, 就是实现一个两级的优先队列.

---

## 练习2: 实现 Stride Scheduling 调度算法 (需要编码) 

### 设计实现过程

详细过程参见 `kern/schedule/stride_default_sched.c`.
为了编码简便, 我忽略了 `USE_SKEW_HEAP` 这一开关, 直接只实现了使用斜堆的.
当然, 使用链表的在此基础上非常容易添加, 不再赘述.

* `BIG_STRIDE` 的选择: 容易证明, 如果在过去足够长的时间内没有新进程产生, 则已有的所有进程必定满足
`Stride_{max} - Stride_{min} \le Pass_{max} = \frac{\mathtt{BIG_STRIDE}}{Priority_{min}}`.
如果不出现溢出错误, 则应该有 `Pass_{max} < \mathtt{MAX_SIGNED}`.
因为`Priority_{min}` 就是 1, 所以上述不等式即为 `\mathtt{BIG_STRIDE} < \mathtt{MAX_SIGNED}`.

实验中为了保险取 `BIG_STRIDE = 0x3FFFFFFF`.

* Skew Heap 的使用:
空的斜堆直接就是 `NULL`, 不需要初始化.

* 其余设计过程: 基本就是 Round Robin 的复制之后加入斜堆的过程.

如 `init()` 中初始化空斜堆.

出入队时不仅要出入 `run_list` 还要出入 `lab6_run_pool`

选择时直接选择斜堆堆顶 (参见堆的基本性质)

`tick()` 不做修改.

