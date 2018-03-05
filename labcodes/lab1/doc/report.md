# 操作系统 lab1
_计53 戴臻旸 2015011296_

---
## 练习1：理解通过make生成执行文件的过程。
### 1. 操作系统镜像文件ucore.img是如何一步一步生成的?(需要比较详细地解释Makefile中每一条相关命令和命令参数的含义,以及说明命令导致的结果)

1. `UCOREIMG	:= $(call totarget,ucore.img)`
</br> 这条命令产生 `bin/ucore.img`, 其中 `totarget` 即将 `ucore.img` 前缀以 `bin/`

1. `$(UCOREIMG): $(kernel) $(bootblock)`</br> 具体生成ucore的命令
>
	`$(V)dd if=/dev/zero of=$@ count=10000`
        创建磁盘镜像 ucore.img, 由 10000 个扇区, 每个 512 字节构成</br>
>
	`$(V)dd if=$(bootblock) of=$@ conv=notrunc`</br>
        将 bootloader (512字节大小) 代码放入 ucore.img 最开头</br>
>
	`$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc`</br>
        将 kernel 代码放入 ucore.img 从 ucore 第二个扇区开始的地方. 
        因为一个扇区是512字节, 所以就相当于将 kernel 在 ucore.img 紧挨着 bootloader 放置.
        符合课上的描述, 即磁盘的前 512 字节是引导记录, 操作系统代码紧挨其后

1. `$(kernel): $(KOBJS)`</br>
	 生成操作系统的代码, 编译链接`kern/`和`lib/`中代码的目标文件
>
	`@echo + ld $@`</br>
	屏幕显示链接生成 kernel 的信息
>
	`$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)`</br>
        按照 `tools/kernel.ld`, 将目标文件链接生成 `bin/nernel`
>
	`@$(OBJDUMP) -S $@ > $(call asmfile,kernel)`</br>
        生成 `obj/kernel.asm` 即 kernel 的反汇编结果
>
	`@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)` </br>
        生成 `obj/kernel.sym` 符号表

1. `$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)` </br>
	生成 bootloader 代码, 包含 `boot/` 中代码;</br>
	并且由 `tools/sign.c` 生成 `bin/sign` 可执行文件用来生成最终的引导记录
>
	`@echo + ld $@` </br>
	同上, 显示链接得到 bootloader 的信息
>
	`$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)`</br>
        将 bootloader 的目标文件链接
>
	`@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)`</br>
        生成 `obj/bootblock.asm` 方便调试
>
	`@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)` </br>
        拷贝一个纯净的 bootloader (即只包含代码指令的)
>
	`@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)`</br>
        使用 `bin/sign` 将 bootloader 代码转换成能放到引导扇区中的格式 
            (事实上就是加一个末尾的0x55, 0xAA)

### 2. 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么?
    大小合法即可, 即不包含结尾的 0x55, 0xAA 时大小不超过 510 字节即可

---
## 练习2：使用qemu执行并调试lab1中的软件。
* 需要稍微修改一下 makefile 中的 `debug` 项目. 因为原来是重新启动一个 term, 但是这个 term 的工作目录就在 `$HOME`了, 而不是 `lab1/` 了. 

        debug: $(UCOREIMG)
            $(V)$(QEMU) -S -s -parallel stdio -hda $< -serial null &
            $(V)sleep 2
            gdb -q -x tools/gdbinit       

* 之后修改 `tools/gdbinit` 为

        file bin/kernel
        target remote :1234
        break *0x7c00
        continue

其中 `break *0x7c00` 就是在 BIOS 的第一条指令处设置断点.

* 之后执行 `make debug`

* 在跳出的 gdb 窗口中, 输入 `x /20i $pc` 即可打印从 0x7c00 开始的 20 条指令

* 之后可以输入 `b *0x7d1a` 就可以跳到 `bootmain()` 了 (地址是从 `obj/bootblock.asm` 中看到的). 然后再执行 `x \20i $pc` 可以打印 `bootmain()` 中的指令

---
## 练习3：分析bootloader进入保护模式的过程。

### 1. 为何开启A20,以及如何开启A20
    因为我们不需要在启动时使用古老的,
    针对8086写的利用 20 位总线回绕特性的bootloader,
    所以我们要打开 A20 (i.e. 地址不回绕), 方便使用32位地址空间.</br>
    之后因为保护模式也不需要将地址的第21位设为0, 所以我们要保持打开.

    开启过程如下 (以下的写 input buffer 都要等到 8042 的 input buffer为空再做)
        1. 给 8042 的 0x64 端口发送 0xd1, 准备写 output port
        2. 准备好后将 output port 的第2位设为1

### 2. 如何初始化GDT表
在 bootasm.S 中初始化和设定 GDT 表, 具体过程如下

1. 首先其初始化了一块内存, 其中包含

	* [selector=0] 一个空段 (base=0, limit=0)
	* [selector=8] 一个代码段 (base=0, limit=4G, P=1 (段存在), S=1 (代码数据), DPL=0 (内核权限), type=0xA (rx))
	* [selector=0x10] 一个数据段 (base=0, limit=4G, P=1, S=1, DPL=0, type=0x2 (rw))

2. 之后将 [段表大小 (2B), 段表地址 (4B)] 放入内存中, 如

        gdtdesc:
            .word 0x17                                      # sizeof(gdt) - 1
            .long gdt                                       # address gdt

3. 调用 `lgdt gdtdesc` 读取 GDT 表

> TODO: 问题, 为什么要是 sizeof(gdt) 再要减去1, 而不是 sizeof(gdt)?

### 3. 如何使能和进入保护模式
    1. 如上所述, 通过 `lgdt` 读取段表
    2. 设置 CR0.PE 即 CR0 = CR0 | 0x1.

这样就相当于进入保护模式了.
之后再初始化段寄存器, 如

* `ljmp $PROT_MODE_CSEG, $protcseg` 初始化代码段
*       movw $PROT_MODE_DSEG, %ax                       # Our data segment selector
        movw %ax, %ds                                   # -> DS: Data Segment
        ...
	初始化其他段寄存器

---
## 练习4：分析bootloader加载ELF格式的OS的过程。
### 1. bootloader如何读取硬盘扇区的？
1. 等待磁盘就绪 (不停轮询0x1f7端口)

2. 写 1 到 0x1f2 端口, 表示读一个扇区

3. 写 要读取的扇区号 到 0x1f3~0x1f6 端口每个端口写8位. </br>
但 0x1f6端口高 4 位为一些控制信息, 其指定 LBA 模式和使用主盘.

4. 写 0x20 到 0x1f7 端口, 发出读开始指令

4. 等待磁盘就绪 (同上)

5. 从 0x1f0 端口循环读取数据到目标内存地址.


### 2. bootloader是如何加载ELF格式的OS？
bootloader中通过 `readseg(dst_va, n_read_bytes, kernel_elf_offset)` 将 kernel 中从 `kernel_elf_offset` 开始的 `n_read_bytes `读取到内存中 `dst_va` 开始的区域, 具体有如下步骤:

1. 从磁盘开头, 引导记录之后 (扇区号从1开始的区域), 读取一页 (4K, 等于8个扇区),
其中包含了所有program header

2. 检查 ELF Magic, 确定 kernel 格式是 ELF

3. 顺序读取每个 program header 对应的 area

---
## 练习5：实现函数调用堆栈跟踪函数
结果如图
![stack trace](/home/hob/Programs/courses/operating-systems/ucore_os_lab/labcodes/lab1/doc/lab1.5.png =x300)

最后一行是 `bootmain()` 的栈帧, 其中

* `ebp:0x00007bf8`</br>
	从 `bootasm.S` 中知道, 最开始的时候有设置为 `%esp=0x7c00; %ebp=0x0`, 这时执行 
        push %ebp
	后栈中内容为

              7c00    old %ebp (equals 0)
        (esp) 7bf8    ...             
              7bf4    ...

	再执行

        movl %esp, %ebp

	后栈中内容为

                   7c00    old %ebp (equals 0)
        (esp,ebp)  7bf8    ...             
                   7bf4    ...

* `eip:00007d6e`
    此处的代码是 `((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();`
    正好是离开 `bootmain()` 导致 `%eip` 被保存到栈上的代码

* `args:0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8`
    这是 7c00~7c10 的内存. 这一内存区域是 bootloader 代码开始存放的区域.
    所以这些参数事实上是 bootloader 最初几条指令

* `<unknow>: -- 0x00007d6d --`
    这一项总是和上面的 `eip: xxxxxxxx` 相连的, 所以参见eip的解释.

---
## 练习6：完善中断初始化和处理 
代码参见 `kern/trap/trap.c`.
### 1. 中断描述符表（也可简称为保护模式下的中断向量表）中一个表项占多少字节？其中哪几位代表中断处理代码的入口？
一个表项占用 8 字节.
最开始 2 字节和最后 2 字节表示入口在段内偏移, 第3和第4字节是段selector, 它们共同指定中断处理程序的入口

---
# 操作系统 lab1 我的答案和参考答案的区别

## 练习1
没有详细解释 `gcc`, `objcopy` 的各个参数具体的意思.

## 练习2
标准答案里还要在 `tools/gdbinit` 中执行 `set arch i8086`.
但是我认为这不是必要的, 因为 x86 是向前兼容的.
事实上就我的实验结果, 就算不加这一句话, 也没有发现问题.

## 练习3
略过了最开始的初始化, 如关中断和清除DF (为了后来的读扇区使用串指令)

## 练习6
* 我硬编码了 `idt[]` 的长度 256, 但是标准代码使用 `sizeof(idt) / sizeof(struct gatedesc);`
* 对于 `SETGATE(...)`, 标准答案所有的 `istrap` 都等于 0, 而我的是 `istrap=(intrno<32)` 即中断号小于 32 时认为是异常 (这一点是从实验指导书上看到的)
* `SETGATE` 的 `DPL`, 我的是对于 `T_SYSCALL=0x80` 为用户权限, 而标准答案是 `T_SWITCH_TOK=121`
* 标准答案的 `ticks` 一直增加而只在判断的时候取模, 可能发生整数溢出.  而 `uint32_t` 的上界不是 100 的倍数, 可能在极少数情况 (500天左右一次) 下出现一次打印 ticks 的时间间隔突然变小的情况. 不过对于我们的项目, 这无伤大雅.