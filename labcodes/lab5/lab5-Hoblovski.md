# 操作系统 lab5

## 练习0: 填写已有实验
这里我改正了之前的一些错误

* 打印 ticks 时我以前是自己在 `trap/trap.c` 中新建了一个变量 `static int n_ticks;`.
但是一个偶然的机会我发现本项目中已经有声明过一个专用的 `ticks` 变量 (`driver/clock.c`)
为了保证和已存项目一致, 我去掉了自己的 `n_ticks` 而是用了自带的 `ticks`.

* 我发现 LAB1 中自己的判断 DPL 写反了, 本来是只有 syscall 才能在用户态调用,
结果写错变成除了 syscall 都能在用户态调用.

---

## 练习1: 加载应用程序并执行(需要编码)

### 设计实现
基本按照注释实现即可.

通过阅读代码容易看出, 已有的 `load_icode()` 已经完成了如解析 ELF, 加载入内存, 初始化 .bss 等工作
我们只需要设置返回时的 trapframe, 使得程序从内核态返回时能跳转到新程序的开始.

为此, 需要修改诸段寄存器 (修改到用户模式的段选择子), 重设用户栈 (esp), 
指明控制流 (eip 指向用户程序代码的起始地址), 开中断 (否则无法利用时钟中断调度).
至于 trapframe 中剩下的, 只有通用寄存器, 显然这些不用理会.

### 请在实验报告中描述当创建一个用户态进程并加载了应用程序后, CPU是如何让这个应用程序最终在用户态执行起来的。
即这个用户态进程被ucore选择占用CPU执行(RUNNING态), 到具体执行应用程序第一条指令的整个经过。

1. 通过 fork 系统调用, 这个进程被创建 (`do_fork()`), 是某个进程的一个 "拷贝"

1. 这个进程被调度器选择, `proc_run(next)`

1. 在 `proc_run()` 中, 其完成地址空间的切换 (CR3), 新的内核栈, 然后 `switch_to()`

1. `switch_to()` 利用 `proc->context`, 设置基本的信息, 包括转移控制流, 开始运行这个进程

1. 这个进程是子进程, `fork()` 返回 0, 被某个检查发现, 之后有 execve 系统调用

1. execve 中最终到 `do_execve()` 中, 其中解析 ELF 文件, 设置内存等等,
最后设置 `trapframe` 使得从系统调用返回时, 跳到的是被加载的程序的第一条指令

---

## 练习2: 父进程复制自己的内存空间给子进程(需要编码)

### 设计实现
基本按照注释实现即可. 需要注意时钟中断处理程序中不再需要 `print_ticks()`,
因为它会在 `make grade` 时提前终止程序.

完成内存复制的就是 `mm/pmm.c:copy_range()` 函数, 阅读代码发现我们只需要完成两件事

1. 复制内存内容. 这个已经有两个 `struct Page` 了, 直接复制页就好了.
2. 设定子进程中虚拟地址映射. 我们复制的源是父进程地址空间的 start,
因此指定子进程的虚地址 start 映射到上一条的那一页.

### 简要说明如何设计实现”CopyonWrite机制“,给出概要设计

在 `do_pgfault()` 中检查, 假设 `P` 产生了 page fault.

Copy on Write 的唯一可能就是, 父子共用的页, `P` 本身是只读的,
但是在进程控制中, `P` 是可写的. 这种矛盾是 COW 的唯一可能.

在 `do_pgfault()` 中判断这种可能, 如果遇到了这种情况, 则

1. 新建一个页, 将 `P` 的内容复制过去

2. 修改页表项 `*ptep`

---

## 练习3: 阅读分析源代码,理解进程执行 fork/exec/wait/exit的实现,以及系统调用的实现(不需要编码)

### 执行 fork 等的表现

* fork: "called once, return twice" -- csapp. 调用之后,
出现两个进程, 看起来是出现了两个一样的程序, 都从 fork() 中返回.
只有返回值有区别.

* exec: "no return". 调用之后, 这个进程就像被换壳一样,
其内容被新的程序数据代替, 就像重新执行另外那个程序一样.

* wait: 阻塞调用, 阻塞到 pid 为参数的子进程完成,
至此这个子进程的所有资源都被回收.
wait 还能让父进程了解子进程的返回值.

* exit: "no return". 调用之后,
当前进程直接结束 (变成僵尸, 直到被父进程或者 init 给降服掉),
其参数是这个进程的返回值.

### 系统调用的实现

在 ucore 系统中, 系统调用是通过软中断 `int $0x80` 实现的.
不同的系统调用通过系统调用的参数区别, 而参数是存放在寄存器中的,
如 eax: 系统调用号, edx, ecx, ebx, edi, esi 是其余 (至多) 5个参数.

每次发生系统调用时, 控制流会从原程序, 到 `__vector`, 到 `trap_dispatch()`,
到 `syscall()`. 在 `syscall()` 中按照调用号分派处理程序,
处理程序之后就可以按照自己的方式处理系统调用了, 如调用其他函数等等.

处理完后, 唯一的返回值被存放在 trapframe 中的 eax, 它会在 `__trapret`
中被弹出到 eax 寄存器中, 到用户程序中就得到了这个返回值.
当然唯一的返回值不意味着唯一的传递信息方式, 还可以有写内存等方式, 如 `read()`.
