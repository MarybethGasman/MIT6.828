# Lab: page tables
> Some operating systems (e.g., Linux) speed up certain system calls by sharing data in a read-only region between userspace and the kernel. This eliminates the need for kernel crossings when performing these system calls. To help you learn how to insert mappings into a page table, your first task is to implement this optimization for the getpid() system call in xv6.

简单介绍一下实验背景，我们知道用户态和内核态，通过一些特殊的指令中断进行切换，切换的时候有一些开销[1]。很多时候我们需要避开这些额外的开销以获得更好的性能。

## 一、进程地址空间分布

![Alt text](img/Screenshot%20from%202023-03-04%2007-30-31.png)

以及[memlayout.h的注释](https://github.com/MarybethGasman/xv6-labs-2021/blob/pgtbl/kernel/memlayout.h#L62-L71)

记住这个USYSCALL  
## 二、加速系统调用

如果我们需要实现一个getpid系统调用，一种方法是内核从proc结构体中取出pid返回给用户程序，[查看代码](https://github.com/MarybethGasman/xv6-labs-2021/blob/pgtbl/kernel/sysproc.c#L23)。  
这么做有内核态用户态切换开销，如何不用切换到内核态完成这个任务？

另一个思路是在进程地址空间创建一个用户态程序只读内核态可写的共享区域，这样获取pid就不用进入内核态，直接访问共享区域就行([查看代码](https://github.com/MarybethGasman/xv6-labs-2021/blob/pgtbl/user/ulib.c#L145-L150))，避免了额外的开销，这块区域就是上面的USYSCALL。


> When each process is created, map one read-only page at USYSCALL (a VA defined in memlayout.h). At the start of this page, store a struct usyscall (also defined in memlayout.h), and initialize it to store the PID of the current process. For this lab, ugetpid() has been provided on the userspace side and will automatically use the USYSCALL mapping. You will receive full credit for this part of the lab if the ugetpid test case passes when running pgtbltest. 

注意到USYSCALL是虚拟地址，内核只需要在进程创建的时候将这个虚拟地址映射到p->usyscall即可

根据[代码](https://github.com/MarybethGasman/xv6-labs-2021/blob/pgtbl/kernel/vm.c#L134)可以推断p->usyscall是物理地址，这一点我不是很明白，p->usyscall是[kalloc出来的](https://github.com/MarybethGasman/xv6-labs-2021/commit/c38c84b49f281c7a080acf44f40c782d12994364#diff-f06ba4e6ae5b13cb787bad05af2c231bcf826410f25bfd78b18774115f0d20a8R144)，内核既使用虚拟地址又使用物理地址还挺难理解它怎么做的[4]。

讲这么多大概就够了，完整的代码[在这里](https://github.com/MarybethGasman/xv6-labs-2021/commit/c38c84b49f281c7a080acf44f40c782d12994364)

## 三，打印页表

现代操作系统一般使用多级页表进行内存管理

>Define a function called vmprint(). It should take a pagetable_t argument, and print that pagetable in the format described below. Insert if(p->pid==1) vmprint(p->pagetable) in exec.c just before the return argc, to print the first process's page table. You receive full credit for this part of the lab if you pass the pte printout test of make grade. 

# 参考文献
[1] 系统调用开销 https://stackoverflow.com/questions/22732433/unix-system-call-overhead

[2] trampoline https://en.wikipedia.org/wiki/Trampoline_(computing)

[3] xv6-risc-v手册 https://pdos.csail.mit.edu/6.S081/2020/xv6/book-riscv-rev1.pdf


# 拓展阅读

关于内核自己的内存是如何分布的，有两个拓展阅读，[阅读一](https://unix.stackexchange.com/questions/512849/whats-inside-the-kernel-part-of-virtual-memory-of-64-bit-linux-processes)，[阅读二](https://www.kernel.org/doc/html/latest/x86/x86_64/mm.html)