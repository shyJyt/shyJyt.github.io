---
title: Lab4 - traps
date: 2024-04-27 10:23:00
updated: 2024-04-27 10:23:00
categories:
- xv6-labs-2021
---

为什么处理时钟中断过程中，已经有了 trapframe 还需要保存一次上下文？时钟中断处理程序执行过程中会修改 trapframe 中的值？

> 首先需要明确两点：
>
> 1、alarm_handler 在用户态执行
>
> 2、每次 trap（系统调用、中断、异常）都会刷新 trapframe 的内容，使其为当前进程的最新上下文
>
> 如果不另存一份调用 alarm_handler 之前的上下文，在 alarm_handler 使用系统调用 sys_sigreturn() 或者 alarm_handler 执行时间较长再次发生时钟中断时，都会因为 trap 而将现在的进程上下文刷入 trapframe 中，无法恢复到 alarm_handler 执行前的状态。

### make grade

![image-20240428110052525](image-20240428110052525.png)
