---
title: virtualization
date: 2024-04-27 10:26:05
updated: 2024-04-27 10:26:09
categories:
- OS
---

### fork —— 创建新进程

完整的进程状态机复制，包括缓冲区（一块地址空间而已）等等。如果程序的缓冲区未及时清空，就会出现神奇的现象。

```c
#include <unistd.h>
#include <stdio.h>

int main() {
    for (int i = 0; i < 2; i++) {
        fork();
        printf("Hello\n");
    }
}
```

![image-20240429172703992](image-20240429172703992.png)

为了减少 write 系统调用的次数，printf 采用以下三种缓冲策略：

- unbuffered（不缓冲，来一个输出一个）
- line buffered（常见于终端，遇到换行符则刷新）
- fully buffered（输出重定向或者管道）

正是由于采用了不同的策略导致 fork 的时候是否会将未及时输出的 Hello 复制一份给子进程。

### execve —— 加载可执行文件

ELF 可执行文件描述了进程的初始状态，加载可执行文件就是将进程的状态重置为可执行文件的状态。







为什么要有第一个 init 进程？？



跨平台的应用程序需要考虑 OS 和体系结构的不同组合吗？

OS 的实现会跟着体系结构走，按道理只用考虑体系结构就行了



为什么说  NVIDIA 的 Linux 驱动不好？
