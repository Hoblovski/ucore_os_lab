# 操作系统 lab4

## 练习1: 分配并初始化一个进程控制块 (需要编码)

### 设计实现过程
可以通过阅读代码得到,
这里的分配并初始化只是完成了把得到的 PCB 中各个域置为初始值的功能.
其既不完成内存分配, 也不完成初始化内存.
因此直接将各个域置为无效值即可, 如
```C
state        = PROC_UNINIT;           // 才分配 PCB, 肯定没有初始化
pid          = -1;                    // 未指定 pid
runs         = 0;                     // 一次都没运行过
kstack       = NULL;                  // 还没分配内核栈
need_resched = 0;                     // 进程本身还没发话说 "我不用 CPU" 了, 那就保留 CPU 给它
parent       = NULL;                  // 才创建还没指定
mm           = NULL;                  // LAB4 里面一直是 NULL
tf           = NULL;                  // 在 copy_thread() 里面初始化
cr3          = boot_cr3;              // 内核线程共享内核地址空间
flags        = 0;                     // 计算机科学自古以来的惯例
memset(&context, 0, sizeof(context)); // 同上
memset(name, 0, sizeof(name));        // 同楼上
```

### 请说明 `proc_struct` 中 `struct context context` 和 `struct trapframe *tf` 成员变量含义和在本实验中的作用是啥? (提示通过看代码和编程调试可以判断出来)

1. `struct context context` 其中域都是 寄存器的值, 主要在 `switch(struct context* from, struct context* to)` 中被使用.
其作用是完成进程 (线程) 切换, 保存通用寄存器 (这里包括栈寄存器), 转移控制流.

2. `struct trapframe *tf` 的意义在于, 每个进程都有自己的异常中断帧. 另外, 进程被创建之后,
第一次运行时会跳转到 `trapentry.S:forkret`, 其中读取我们的 `tf`,
来设置新进程和父进程的几乎一样的运行流 / 寄存器 etc (但是返回值 `eax =0`, 并且开中断).

---

## 练习2: 为新创建的内核线程分配资源 (需要编码)

### 实现设计过程
基本就按照注释, 一步一步的填空. 注意鲁棒编程, 检查返回值.

1. 调用 `alloc_proc()`, 如果成功, 设置 `pid` 和 `parent` (以后的函数如哈希会使用)

2. 设置 `kstack`, 通过调用 `setup_kstack()`

3. 设置 `mm`, 通过调用 `copy_mm()`

4. 设置 `tf` 和 `context`, 通过调用 `copy_thread`.
这里其实 `esp` 直接传入 `stack` 就好, 因为父进程和子进程栈也是一样的 
(不得不说这里命名非常有误导性).

5. 将这个初始化了一半的进程加入进程表里 (哈希表和链表), 注意维护 `nr_process`

6. 设置 `state` 为就绪, 通过调用 `wakeup_proc()`

7. 设置返回值. 父进程中, 返回值是子进程的 pid. 
(子进程中的返回值由 `copy_thread()` 中一句 `proc->tf->tf_regs.reg_eax = 0;` 置为了 0)

### 请说明ucore是否做到给每个新fork的线程一个唯一的id?请说明你的分析和理由。
在 `get_pid()` 函数中, 我们可以这样理解这个函数的执行流程:
* 维护一个窗口 [`last_pid`, `next_safe`), 初始这个窗口是空的,
这个窗口在函数多次调用中保持不变 (i.e. 函数中 static 变量).
注意这里的变量名并不真实的反映其意义.

每次分配 pid 时,

1. 检查这个窗口是否非空, 若为空则初始化窗口为整个可能的 pid 空间

2. 迭代检查进程表中所有进程 `proc`, 如果 `proc->pid` 落在窗口中, 则需要更新窗口
    1. `proc->pid` != `last_pid`. 则将窗口右界 `next_safe` 置为 `proc->pid`
    2    if (proc != current) {
        bool intr_flag;
        struct proc_struct *prev = current, *next = proc;
        local_intr_save(intr_flag);
        {
            current = proc;
            load_esp0(next->kstack + KSTACKSIZE);
            lcr3(next->cr3);
            switch_to(&(prev->context), &(next->context));
        }
        local_intr_restore(intr_flag);
    }
. `proc->pid` == `last_pid`. 则将窗口左界 `last_pid` 自增
    3. 更新窗口后, 如果窗口为空, 需要重设窗口, 重头开始迭代

3. 迭代完成后一定有窗口非空, 此时直接取窗口左界返回即可

显然, 通过归纳容易证明在执行中任何迭代的一步完成后, 任何已知的 `proc->pid` 都不会落在窗口中.
到第3步时, 即有返回结果不等于任何已有的 pid.

### 和标准答案的区别
标准答案在第5步临时禁用了中断, 为了保证哈希表和进程表的一致性.
5. 将这个初始化了一半的进程加入进程表里 (哈希表和链表), 注意维护 `nr_process`

已改正.

---

##练习3: 阅读代码,理解 `proc_run` 函数和它调用的函数如何完成进程切换的。(无编码工作)

首先, 进程切换只发生在不同的进程中. 在此基础上, 考虑如下过程.

1. 修改 `current` 指针, 指向新的进程

2. 修改 TSS 中的数据. 因为 ucore 使用软件来完成上下文切换, 我们只填充 esp0
    (考虑到 ucore 内核的栈都是一个段所以不修改 ss0).
事实上到了21世纪以后已经没有人使用 Intel 自带的 TSS 来完成上下文切换,
甚至似乎 Intel 自己也知道这一点而放弃了对硬件上下文切换的优化,
只是作为历史包袱而留在那里.

3. 修改页表基址, 通过修改 CR3 寄存器

4. 调用 `switch_to` 切换 (其中就是保存恢复了一些寄存器, 然后转移控制流)

注意这一过程是关中断的.

### 在本实验的执行过程中,创建且运行了几个内核线程?
三个. 一个 `initproc`, 一个 `idleproc`,  
还要包含在此之外的 kernel 代码运行 (即 `schedule()` 前的代码).

虽然其既不是多道程序中的一道 (不受调度控制) 也没有明确的 PCB,
但是从 "一个运行程序的实例",  "运行状态的的抽象" 定义来看, 也能算做一个线程 / 进程.

### 语句 `local_intr_save(intr_flag);....local_intr_restore(intr_flag);` 在这里有何作用? 请说明理由
保证执行的一致性, 即上下文切换的原子性.
比如说, 上下文切换到一般, 刚刚修改了 CR3, 就发生了中断,
那么对于当前进程这个 CR3 是无效的, 这样就会使用错误的页表.
