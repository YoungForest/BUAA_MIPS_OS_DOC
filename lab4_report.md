# 思考题

## Thinking 4.1
    思考下面的问题，并对这两个问题谈谈你的理解：
    • 子进程完全按照fork() 之后父进程的代码执行，说明了什么？
    • 但是子进程却没有执行fork() 之前父进程的代码，又说明了什么？

1. 说明父子进程的`User text`，即可执行代码是完全一样的。
2. 说明子进程的pc值大于等于调用fork()的地址(刚开始我认为子进程的pc值是复制父进程的，看到指导书的后面发现，并不是这样的，子进程的pc是置为异常发生时受害指令(CP0_EPC)的位置的)。

## Thinking 4.2
    关于fork 函数的两个返回值，下面说法正确的是：
        A、fork 在父进程中被调用两次，产生两个返回值
        B、fork 在两个进程中分别被调用一次，产生两个不同的返回值
        C、fork 只在父进程中被调用了一次，在两个进程中各产生一个返回值
        D、fork 只在子进程中被调用了一次，在两个进程中各产生一个返回值

C；

父进程调用fork(),在函数体中生成子进程，子进程通过设置R寄存器(在MPIS R3000中就是v0寄存器)，达到返回不同值的目的。

## Thinking 4.3
    如果仔细阅读上述这一段话, 你应该可以发现, 我们并不是对所有的
    用户空间页都使用duppage 进行了保护。那么究竟哪些用户空间页可以保护，哪些
    不可以呢，请结合include/mmu.h 里的内存布局图谈谈你的看法。

需要对异常栈(UXSTACKTOP-BY2PG)以下的用户空间页进行保护。

因为UTOP至ULIM的3*PDMAP的用户空间是用户态只读的，用户进程不可写，所以不需要保护；

异常栈是用来处理异常的，如果进行保护的话，在触发page fault异常时进入异常处理函数时，可定会用到异常栈，
如果异常栈被保护，又会触发新的异常，造成死循环。
所以，异常栈不需要保护，也不能保护。

## Thinking 4.4
    请结合代码与示意图, 回答以下两个问题：
    1、vpt 和vpd 宏的作用是什么, 如何使用它们？
    2、它们出现的背景是什么? 如果让你在lab2 中要实现同样的功能, 可以怎么写？

1. 作用：根据虚拟地址获得对应的页表项和页目录项。

    在`user/entry.S`中有这样的代码，设置了vpt和vpd:

            .globl vpt
        vpt:
            .word UVPT

            .globl vpd
        vpd:
            .word (UVPT+(UVPT>>12)*4)

    可见，vpt是用户空间的页表的起始位置，而vpd是用户空间的页目录的起始位置(根据页目录的自映射机制计算而来)。

    使用方法：设虚拟地址为va,其页表项为(\*vpt)[va>>12],页目录项为(\*vpt)[va>>22]。

2. 背景：为了使用户进程可以访问页表和页目录。

    在lab2中，由于我们处于 内核态，所以可以直接通过PDX(va)和pgdir寻得页表和页目录。


# 实验难点
1. 最难的是会跳进自己以前挖的坑出不来。

2. fork中pgfault也比较难，不知道映射一个新的页到“临时的位置”中临时的位置是在哪？PPT中说的“PFTEMP”，在我们的实验中也不知所踪。
参照JOS中PFTEMP的位置，它把PFTEMP放在了`UTEXT-PGSIZE`这个地方，我试着在我们实验中也这样做了，将“临时的位置”设为`UTEXT-BY2PG`
,果然成功了。后来和同学们交流后得知，这个“临时的位置”还是很随意的，并没有要求，很多人把其设为了`ROUNDDOWN(0X5000 0000 - BY2PG)`这个位置，他们也并没有充足的理由，也是通过参考其他网上代码获得的位置。

3. 比较坑的一点是PPT和代码注释并不是都靠谱的，指导书还比较靠谱，但直接有一小节没有的样子。

# 残留疑点
1. 在函数`int sys_set_pgfault_handler(int sysno, u_int envid, u_int func, u_int xstacktop)`中有这样的前置条件：`Pre-Condition:xstacktop points one byte past exception stack. `
我把传入的`xstacktop`输出发现，它的值恰好等于`UXSTACKTOP 0x7f40 0000`.
根据注释的前置条件，它应该等于`UXSTACKTOP+1`啊.我认为应该等于`UXSTACKTOP 0x7f40 0000`，是注释写错了。

2. sys_mem_alloc和sys_mem_map两个函数中的pre-condition关于perm的要求，我在函数体中进行了判断，但发现并不是都满足。
所以我对注释中的前置条件的保证比较怀疑。

3. env.c中load_icode_mapper()函数中，我在映射新分配的页时，设置的权限位没有PTE_R，只有PTE_V，但lab4的测试程序会发生“pageout TOO LOW”，只有加上PTE_R之后才好。
复制进程的可执行代码到内存中，我认为进程是不应该有可写权限的，进程不能修改自己的可执行代码，否则会带来不安全的问题。
但我也不清楚是哪里出了问题。

# 感想与体会
lab4的实验与之前相比难了很多，可能是因为指导书和代码注释的不完整导致的，看来OS实验还有很多有待完善的地方。
一个很大的感受是，注释是不可信的，自己之前填的函数也是不可信的。

在完成实验的过程中，我遇到了很多“蜜汁错误”，之前实验的不小心现在也吃了苦头，有些错误是源于之前实验，我甚至都更正了之前填的env.c才获得正确的结果。

感觉自己的调试技巧进步了很多(没办法，被逼的)。除了一直用的很爽的printf(writef)之外，我也学会了看函数调用栈、输出异常。效率提高了很多。