---
title: Lab3 - Page tables
date: 2024-04-25 21:17:31
updated: 2024-04-25 21:17:31
categories:
- xv6-labs-2021
---

虚拟地址和物理地址的转换在什么时候发生？

> 物理内存是指 DRAM 中的存储单元。物理内存的一个字节有一个地址，称为物理地址。指令只使用虚拟地址，分页硬件将其转换为物理地址，然后发送到 DRAM 硬件以读取或写入存储。

操作系统是如何分页硬件沟通的？

>内核必须将根页表页的物理地址写入 satp 寄存器。每个 CPU 都有自己的 satp, CPU 将使用自己的satp指向的页表，转换后续指令产生的所有地址。每个CPU都有自己的satp，使得不同的CPU可以运行不同的进程，每个进程都有一个私有地址空间，由自己的页表描述。通常，内核将所有物理内存映射到其页表中，这样就可以使用加载/存储指令读写物理内存中的任何位置。由于页目录在物理内存中，内核可以使用标准的存储指令向页目录的虚拟地址写入数据，从而对页目录中页目录的内容进行编程。

内核空间和进程的内核栈有什么关系？



解析一个 va 的完整流程



### Speed up system calls

### 思考题1

### 思考题2



作为页表页的物理页没有特殊标记吗？通过深度控制页表层次灵活性不够。



一个地址的 uint64 和 uint64 * 的不同用法

指针赋予了变量进行地址运算的能力。



### 如何用  gdb 调试 xv6

比如说在第一个实验中，出现 panic: freewalk leaf ，如何找到错误？

先在实验根目录 `make qemu-gdb` 启动 qemu 调试模式，再在另一个终端 `gdb-multiarch -x .gdbinit` 启动 gdb 并以 .gdbinit 文件配置的内容初始化。

```shell
# .gdbinit
set confirm off
set architecture riscv:rv64
target remote 127.0.0.1:26000
# 设定符号文件为 kernel，即编译后的内核可执行文件
symbol-file kernel/kernel
set disassemble-next-line auto
set riscv use-compressed-breakpoints yes
```

kernel/kernel 中包含了所有内核函数，可通过函数名下断点，其更具体的内容在 kernel.asm 中。继续执行后可观察到 qemu 启动，剩余部分就是常规的 gdb 调试了。



为什么在释放页表时需要解除映射？

```C
void
proc_freepagetable(pagetable_t pagetable, uint64 sz)
{
  uvmunmap(pagetable, TRAMPOLINE, 1, 0);
  uvmunmap(pagetable, TRAPFRAME, 1, 0);

  #ifdef LAB_PGTBL
  uvmunmap(pagetable, USYSCALL, 1, 0);
  #endif
  
  uvmfree(pagetable, sz);
}
```





PTE_* 是在什么时候被设置的？为什么 PTE_A 在访问完后要清除？



### make grade

![image-20240426225954867](image-20240426225954867.png)
