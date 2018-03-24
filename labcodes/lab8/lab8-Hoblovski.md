# 操作系统 lab8

## 练习1: 完成读文件操作的实现 (需要编码)
事实上这里包含读和写. 为了简便讨论读即可.

详细设计参见代码 `kern/fs/sfs/sfs_inode.c`.

### 简要设计
用到了函数包括
* `sfs_bmap_load_nolock(sfs_struct, sfs_inode, logical_blkno, *physical_blkno)` 完成了逻辑块号到物理块号的转换

* `sfs_buf_op(sfs_struct, buf, len, physical_blkno, offset)` 将物理块中 `offset` 开始 `len` 个字节读入 `buf` 中.
注意 `offset` 开始 `len` 个字节必须都在 `physical_blkno` 的这个物理块中.

* `sfs_block_op(sfs_struct, buf, physical_blkno, nblks)` 将从 `physical_blkno` 开始的 `nblks` 个物理块的内容读入 `buf` 中.

基本设计是, 对于最开始和最后一个块, 读可能是没有按照块对齐的, 因此使用 `sfs_buf_op()` 读入.
当然 `sfs_buf_op()` 也能读入一个完整的块 i.e. 可以处理对齐读入的情况.

而中间的块, 使用 `sfs_block_op()` 读入, 但是一次只能读入一个块 (`nblks` 为 1), 因为 `nblks` 指的是连续物理块的块数,
而中间的块是连续的*逻辑*块.

于是显然有三个步骤
1. 读第一块
2. 如果中间有整块需要读入, 读中间的块
3. 如果最后一块不是第一块, 处理最后一块

### 和标准答案的区别
对于函数的返回值为错误情况, 我使用的 `assert` 而标准答案直接中断读入.

### UNIX的PIPE机制

通过阅读 `user/sh.c` 容易看出, 两个进程之间的管道是由其父亲进程实现的.
为方便假设一个例子 `$ producer | consumer`.

`sh` 调用 `pipe()` 得到两个文件描述符,
将 `producer` 的 stdout 利用 `dup2()` 复制到第一个,
将 `consumer` 的 stdin 用 `dup2()` 被重定向到第二个,
之后关闭 `producer` 的默认 stdout (`FILENO_STDOUT=1`) 和 `consumer` 的默认 stdin.

因此可以设计一个新的文件系统 pipefs, 其中设计类似 sfs,
只是不将操作写到磁盘上, 而是写到内核中某段预留的内存中.
之后读写都在这段内存中进行.

---

## 练习2: 完成基于文件系统的执行程序机制的实现 (需要编码)
详细实现参考 `process/proc.c` 文件.

### 简要设计
首先需要为进程加入打开文件表, 这只需要修改 `struct proc->filesp` 即可.
就是在 `alloc_proc()` 时初始化一个空的文件表, 在 `do_fork()` 中子线程继承父线程打开的文件 (调用 `copy_files()`) 即可.

主要是 `load_icode()` 函数. 之前的实验中 `load_icode()` 是从一个内存位置读 ELF, 而这一次需要从文件中读 ELF.
实现时基本和之前的 `load_icode()` 类似, 但是有如下区别

1. 需要将所有的读内存的操作转换成读文件的操作.

2. 之前的 `load_icode()` 中没有对 `argc`, `argv` 的支持, 这里需要添加. 即在用户栈的顶部放入 `argv` 和 `argc`.

我卡在了实现用户栈的部分, 后来参考答案才发现用户栈设立之后需要有一个对齐操作.

### 和标准答案的区别
* 标准答案读入文件到栈内存, 而我读入到堆内存 (当然在最后有清理).

* 我没有关闭 `fd`. 事实上 `fd` 在 `do_execve()` 中打开, 在 `do_execve()` 中没有关闭. 因此应该在 `load_icode()` 中关闭.

* 标准答案中开中断是 `tf->tf_eflags = FL_IF;` 而我是 `tf->tf_eflags |= FL_IF;`. 根据 fork 语义, 后者应该是正确的.

* 标准答案中检查了 `argc` 大小, 以及诸 `argv[i]` 的长度.
不过事实上这些检查都在 `do_execve()` 中和 `copy_kargv()` 中完成了.

### UNIX的硬链接和软链接机制

1. 硬链接.
硬链接就是一个普通文件, 但是它的 `inode` 是和其他文件共享的 i.e. 有多个文件使用这个 `inode`.
</br>
首先需要实现 SFS 中的 `vop_create(dir, name, excl, node)`.
事实上对于硬链接我们只需要实现 `node` 是一个已有的 `inode` 即可.
这一步从根目录 `/` 开始使用 `vfs_lookup()` 函数寻找, 将已有的 `inode` 加入目录内容即可.
</br>
之后在 `vfs_link` 中直接寻找到 `oldlink` 的 `inode`,
利用 `vop_create()` 创建 `newlink` 文件,
其 `inode` 和 `oldlink` 的是一样的即可.

2. 软链接
软链接应当成为一种新的文件类型 (regular file, directory 以外另一种文件).
其中内容只需包含其指向的文件的路径即可.
(类型就是 `sfs_disk_inode->type` 中类型)

之后对于这种类型的文件, 新建一系列函数, 其中内容很简单就是
1. 读取文件中的路径
2. 将请求转发给那个路径的被链接的文件
之后在 `fs/sfs/sfs_inode.c` 最末尾加入
```C
// The sfs specific SYMBOLIC-LINK operations correspond to the abstract operations on a inode.
static const struct inode_ops sfs_node_slinkops = {
    .vop_magic                      = VOP_MAGIC,
    .vop_open                       = // sfs_openslink
    // ... all other sorts of similar stuff
};

