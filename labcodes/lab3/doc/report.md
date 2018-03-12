# 操作系统 lab3

## 练习1: 给未被映射的地址映射上物理页 (需要编程)

### 简要设计
在 pagefault handler 中, 首先检查地址对应的页表项.

如果为0, 因为我们交换分区的扇区号是从 1 开始的,
那么说明这是一个空页表项, 因此直接分配一页物理页即可.

否则参见练习2.

### 请描述页目录项(PageDirectorEntry)和页表(PageTableEntry)中组成部分对ucore实现页替换算法的潜在用处。
* Present 位: 确定是否替换 (所有 P 位都为 1, 才需要替换)

* Dirty 位: 采用回写时, 对于没有修改的页可以不用写磁盘, 减小开销

并且, 当 Present 位为 0 时, 前 24 位还索引了交换分区中的扇区号.

### 如果ucore的缺页服务例程在执行过程中访问内存,出现了页访问异常,请问硬件要做哪些事情?
* 将 CR2 寄存器的值设为导致 pagefault 的线性地址

* 设置异常号, 保存现场 ... 跳转到异常处理入口. 即一般异常的处理过程.

* 将 errorCode 放到中断栈上

### 我和标准答案的区别
* 调用 `get_pte()`, `pgdir_alloc_page()` 后, 没有检查返回值为`NULL`的情况

### 其他
在 lab1 中, 我的 `n_tick` 每次达到 100 后就清零了.
实际上他可能用于操作系统中时间相关的操作, 即作为一个计时器, 因此不应清零.

---

## 练习2: 补充完成基于FIFO的页面替换算法 (需要编程)

### 简要设计
所有的驻留页面形成一个链表 (通过 `struct Page.pra_page_link`),
换入内存时插入链表尾部, 换出时取链表头部换出.

在 pagefault handler 中, 如果页表项不为 0, 
说明页表项是索引交换分区中的扇区, 因此使用 `swap_in()` 读入此页,
重设虚实映射, 并且通知 swap manager 即可.


### 如果要在ucore上实现"extendedclock页替换算法"请给你的设计方案,现有的swap manager框架是否足以支持在ucore中实现此算法? 如果是,请给你的设计方案。 如果不是,请给出你的新的扩展和基此扩展的设计方案。
现有框架无法支持, 因为其无法在访问页时修改页的属性,
因此无法做到访问一个页就把它的访问位设为 1, 修改位同理.

因为访存是非常频繁的操作, 不可能由操作系统每次去修改页表项,
所以这里需要硬件支持.
具体说来, 需要 TLB 在每一次地址翻译的时候都修改 TLB 项
(也就是修改页表项, 因为所有被使用的页表项一定在 TLB 中),
包括设置访问位和修改位.

在硬件维护访问修改位的前提下, 不妨假设访问修改位都在 `struct Page.flags` 中,
那么稍微修改 `swap_fifo.[ch]` 就可以完成任务, 如示例代码
```C
// swap_eclock.c

list_entry_t *eclock_clkptr;

// ...

static int
_eclock_map_swappable(struct mm_struct *mm, uintptr_t addr, struct Page *page, int swap_in)
{
    list_entry_t *head = (list_entry_t*) mm->sm_priv;
    list_entry_t *entry = &(page->pra_page_link);
    assert(entry != NULL && head != NULL);

    clear_bit(PTE_ACCESSED, page->flags);
    clear_bit(PTE_DIRTY, page->flags);

    // add to queue rear
    list_add_before(head, entry);
    return 0;
}

static int
_eclock_swap_out_victim(struct mm_struct *mm, struct Page ** ptr_page, int in_tick)
{
    list_entry_t *head = (list_entry_t*) mm->sm_priv;
    assert(head != NULL);

    while (1) { // guaranteed to converge
        struct Page *p = le2page(eclock_clkptr);
        if (!(PTE_ACCESSED & p->flags)) {
            if (!(PTE_DIRTY & p->flags)) {  // A = 0, D = 0
                // don't directly delete eclock_clkptr!
                //  that results in a detatched list!
                eclock_clkptr = list_next(eclock_clkptr);
                list_del(list_prev(eclock_clkptr));
                *ptr_page = p;
                return 0;
            } else {                        // A = 0, D = 1
                clear_bit(PTE_DIRTY, p); 
            }
        } else {
            if (!(PTE_DIRTY & p->flags)) {  // A = 1, D = 0
                clear_bit(PTE_ACCESSED, p);
            } else {                        // A = 1, D = 1
                clear_bit(PTE_ACCESSED, p);
            }
        }
    }
    assert(0);  // never here
}
```

### 并需要回答如下问题
#### 需要被换出的页的特征是什么?
访问位和修改位都为 0.

#### 在ucore中如何判断具有这样特征的页?
直接检查其 `Page` 结构体的 `flags` 即可.

#### 何时进行换入和换出操作?
* 换入: 访问此页面, 但此页面不在内存中时就换入

* 换出: 换入找不到位置的时候换出, 如果积极一点就也周期性换出一些页 
(执行上述算法即可).

### 我和标准答案的区别
* `kern/mm/vmm.c:do_pgfault()` 中,
调用 `swap_map_swappable(mm, addr, page, swap_in)` 时,
我将参数 `swap_in` 设为 0, 标准答案设为1.
事实上我们的 FIFO 置换算法没有使用这个参数, 而且也没有注释说明这个参数的意义.

* 标准答案的队列似乎是反的. 不过这不影响最终结果, 仅是实现者的偏好不同.

