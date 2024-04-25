---
title: Lab2: System calls
date: 2024-04-22 11:34:22
updated: 2024-04-22 11:34:22
categories:
- xv6-labs-2021
---

### 1. xv6 启动流程

​	当 RISC-V 计算机启动时，它初始化自己并运行存储在 ROM 中的引导加载程序。引导加载程序将 xv6 内核（镜像文件）加载到内存中指定位置 0x80000000，此时分页并未启用，虚拟地址直接映射到物理地址。借助 kernel.ld 链接脚本将下列代码放在指定位置，然后，在机器模式下，CPU 从 `_entry ` 开始执行xv6。

```assembly
	# qemu -kernel loads the kernel at 0x80000000
    # and causes each CPU to jump there.
    # kernel.ld causes the following code to
    # be placed at 0x80000000.
.section .text
.global _entry
_entry:
	# set up a stack for C.
        # stack0 is declared in start.c,
        # with a 4096-byte stack per CPU.
        # sp = stack0 + (hartid * 4096)
        la sp, stack0
        li a0, 1024*4
	csrr a1, mhartid
        addi a1, a1, 1
        mul a0, a0, a1
        add sp, sp, a0
	# jump to start() in start.c
        call start
spin:
        j spin

```

​	之所以将内核放置在 0x80000000 而不是 0x0，是因为地址范围 0x0 : 0x80000000 包含了 I / O 设备。`_entry`处的指令建立了一个栈（允许函数调用），使 xv6 可以运行 C 代码。xv6 在文件 start.c 中为初始栈 stack0 声明了空间。现在内核有了栈，开始时调用C代码。

​	start() 函数执行一些只允许在机器模式下执行的配置，然后切换到 supervisor 模式。为了进入 supervisor 模式，RISC-V 提供了 mret 指令。该指令通常用于从上一次调用从 supervisor 模式返回到机器模式。start() 不会从这样的调用中返回，它在寄存器 mstatus 中将之前的特权模式设置为 supervisor，通过将 main() 的地址写入寄存器 mepc 来将返回地址设置为 main，通过将页表寄存器 satp 写入 0 来禁用 supervisor 模式下的虚拟地址转换，并将所有中断和异常委托给 supervisor 模式。在进入 supervisor 模式之前，start 还要时钟芯片进行编程，以产生定时器中断。有了这个清理工作，通过调用 mret 开始“返回”到 supervisor 模式。

```c
// entry.S jumps here in machine mode on stack0.
void
start()
{
  // set M Previous Privilege mode to Supervisor, for mret.
  unsigned long x = r_mstatus();
  x &= ~MSTATUS_MPP_MASK;
  x |= MSTATUS_MPP_S;
  w_mstatus(x);

  // set M Exception Program Counter to main, for mret.
  // requires gcc -mcmodel=medany
  w_mepc((uint64)main);

  // disable paging for now.
  w_satp(0);

  // delegate all interrupts and exceptions to supervisor mode.
  w_medeleg(0xffff);
  w_mideleg(0xffff);
  w_sie(r_sie() | SIE_SEIE | SIE_STIE | SIE_SSIE);

  // configure Physical Memory Protection to give supervisor mode
  // access to all of physical memory.
  w_pmpaddr0(0x3fffffffffffffull);
  w_pmpcfg0(0xf);

  // ask for clock interrupts.
  timerinit();

  // keep each CPU's hartid in its tp register, for cpuid().
  int id = r_mhartid();
  w_tp(id);

  // switch to supervisor mode and jump to main().
  asm volatile("mret");
}
```

​	在main () 初始化几个设备和子系统之后，它调用 userinit() 创建第一个用户态进程。第一个进程执行一个 initcode.S，它调用了 exec 启动了 shell 来代替当前进程。

​	至此启动完成了。

```c
// start() jumps here in supervisor mode on all CPUs.
void
main()
{
  if(cpuid() == 0){
    consoleinit();
    printfinit();
    printf("\n");
    printf("xv6 kernel is booting\n");
    printf("\n");
    kinit();         // physical page allocator
    kvminit();       // create kernel page table
    kvminithart();   // turn on paging
    procinit();      // process table
    trapinit();      // trap vectors
    trapinithart();  // install kernel trap vector
    plicinit();      // set up interrupt controller
    plicinithart();  // ask PLIC for device interrupts
    binit();         // buffer cache
    iinit();         // inode table
    fileinit();      // file table
    virtio_disk_init(); // emulated hard disk
    userinit();      // first user process
    __sync_synchronize();
    started = 1;
  } else {
    while(started == 0)
      ;
    __sync_synchronize();
    printf("hart %d starting\n", cpuid());
    kvminithart();    // turn on paging
    trapinithart();   // install kernel trap vector
    plicinithart();   // ask PLIC for device interrupts
  }

  scheduler();        
}
```

```C
// a user program that calls exec("/init")
// od -t xC initcode
uchar initcode[] = {
  0x17, 0x05, 0x00, 0x00, 0x13, 0x05, 0x45, 0x02,
  0x97, 0x05, 0x00, 0x00, 0x93, 0x85, 0x35, 0x02,
  0x93, 0x08, 0x70, 0x00, 0x73, 0x00, 0x00, 0x00,
  0x93, 0x08, 0x20, 0x00, 0x73, 0x00, 0x00, 0x00,
  0xef, 0xf0, 0x9f, 0xff, 0x2f, 0x69, 0x6e, 0x69,
  0x74, 0x00, 0x00, 0x24, 0x00, 0x00, 0x00, 0x00,
  0x00, 0x00, 0x00, 0x00
};

// Set up first user process.
void
userinit(void)
{
  struct proc *p;

  p = allocproc();
  initproc = p;
  
  // allocate one user page and copy init's instructions
  // and data into it.
  uvminit(p->pagetable, initcode, sizeof(initcode));
  p->sz = PGSIZE;

  // prepare for the very first "return" from kernel to user.
  p->trapframe->epc = 0;      // user program counter
  p->trapframe->sp = PGSIZE;  // user stack pointer

  safestrcpy(p->name, "initcode", sizeof(p->name));
  p->cwd = namei("/");

  p->state = RUNNABLE;

  release(&p->lock);
}
```

```assembly
# Initial process that execs /init.
# This code runs in user space.

#include "syscall.h"

# exec(init, argv)
.globl start
start:
        la a0, init
        la a1, argv
        li a7, SYS_exec
        ecall

# for(;;) exit();
exit:
        li a7, SYS_exit
        ecall
        jal exit

# char init[] = "/init\0";
init:
  .string "/init\0"

# char *argv[] = { init, 0 };
.p2align 2
argv:
  .long init
  .long 0

```



### 2. 系统调用



### 3. make grade

![image-20240422214112132](image-20240422214112132.png)



### 4. 遗留问题

结构体在编译时可以只声明吗？？

> 可以，不过当需要用到它的大小时，比如创建变量、sizeof 等等，就会报错（缺少信息，无法生成中间代码。
>
> ![image-20240422232852951](image-20240422232852951.png)



在声明用户态的 sysinfo() 时为什么需要前向声明 struct sysinfo ？

> 声明指等到链接时再找到具体定义，其实就相当于 #include 所需头文件的一部分，而不是全部导入，由于允许重复声明而不允许重复定义，这样做防止直接 #include 带来的重复定义、循环依赖等问题。

