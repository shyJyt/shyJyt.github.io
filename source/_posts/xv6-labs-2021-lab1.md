---
title: Lab1 - Xv6 and Unix utilities
date: 2024-04-07 13:08:22
updated: 2024-04-08 22:18:00
categories:
- xv6-labs-2021
---
#### 1. 大致内容

熟悉环境以及一些基本的系统调用。

#### 2. read()

##### 2.1. 读管道的几种情况

1. 有东西就读
2. 没东西就阻塞
3. 写端关闭（引用次数为0）且没东西返回 0

#### 3. fork()

##### 3.1. 对于文件描述符是怎么处理的？

父子进程共享文件描述符及其偏移指针，fork() 仅增加引用次数。

```c
// xv6/kernel/proc.c
// increment reference counts on open file descriptors.
for(i = 0; i < NOFILE; i++)
    if(p->ofile[i])
        np->ofile[i] = filedup(p->ofile[i]);
np->cwd = idup(p->cwd);
```

close(fd) 只是减少一次引用，减到 0 才会关闭。

```c
#include <stdio.h>
#include <unistd.h>

int main() {
    int fa[2];
    pipe(fa);

    int tmp = 10;
    write(fa[1], &tmp, sizeof(tmp));

    if (fork() == 0) {
        sleep(5);
        close(fa[1]);
        printf("child wake\n");
    }
    else {
        close(fa[1]);
        
        read(fa[0], &tmp, sizeof(tmp));
        printf("fa read %d\n", tmp);
        int ret = read(fa[0], &tmp, sizeof(tmp));
        printf("ret: %d\n", ret);
    }
    
    return 0;
}

```

输出如下图，且后两行在 5s 后才输出。已知 read 在管道写端被关闭时返回 0 ，推断分别在父子进程 close() 一次后，写端引用次数减到 0 后关闭，read 不再阻塞， 返回 0 .

![image-20240408111553010](image-20240408111553010.png)



#### 4. primes

尽早关闭不用的读 / 写端，避免文件描述符耗尽。

```C
#include "kernel/types.h"
#include "user/user.h"

void solve(int *fa) {
    close(fa[1]);

    int flag;
    read(fa[0], &flag, sizeof(flag));
    printf("prime %d\n", flag);

    int tmp;
    if (read(fa[0], &tmp, sizeof(tmp))) {
        int child[2];
        pipe(child);

        if (fork() == 0) {
            solve(child);
        }
        else {
            close(child[0]);
            if (tmp % flag) {
                write(child[1], &tmp, sizeof(tmp));
            }

            while (read(fa[0], &tmp, 4)) {
                if (tmp % flag) {
                    write(child[1], &tmp, sizeof(tmp));
                }
            }
            close(fa[0]);
            close(child[1]);

            wait(0);
        }
    }

    exit(0);
}

int main(int argc, char *argv[]) {
    int child[2];
    pipe(child);

    if (fork() == 0) {
        solve(child);
    }
    else {
        for (int i = 2; i <= 35; i++) {
            write(child[1], &i, sizeof(i));
        }
        close(child[1]);
        wait(0);
    }

    exit(0);
}
```



#### 5. xargs

```shell
command1 | xargs [options] [command2 [initial-arguments]]
```

作用是将 command1 的输出处理后，使其作为 command2 参数，command2 为可选项，默认为 echo.

实验要求只用实现 -n 1，即一次只从左边取一个参数，而 xargs 的默认参数分隔符为 \n，所以一次取一行作为参数，交由 exec 执行。

```C
#include "kernel/types.h"
#include "kernel/param.h"
#include "kernel/stat.h"
#include "user/user.h"

#define MAXN 1024

int readLine(char *param[MAXARG], int cntp) {
    char buf[MAXN];
    int l = 0, r = 0;
    
    while (read(0, buf + r, 1)) {
        // split by '\n'
        if (buf[r] == '\n') {
            buf[r] = 0;
            break;
        }
        r++;
        if (r == MAXN) {
            fprintf(2, "Too many params!\n");
            exit(1);
        }
    }

    if (r == 0) {
        return -1;
    }

    // split by space
    // ....abc...asd..asd..\0
    while (l < r) {
        while(l < r && buf[l] == ' ') l++;
        param[++cntp] = buf + l;
        while (l < r && buf[l] != ' ') l++;
        buf[l++] = 0;
    }

    return cntp;
}

int main(int argc, char *argv[]) {
    if (argc < 2) {
        printf("Usage: xargs [options] [command [initial-arguments]]\n");
        exit(1);
    }
    
    char *param[MAXARG];

    for (int i = 0; i < argc - 1; i++) {
        param[i] = malloc(strlen(argv[i + 1]) + 1);
		strcpy(param[i], argv[i + 1]);
    }

    int cntp = -1;
    while ((cntp = readLine(param, argc - 2)) != -1) {
        // printf("%s, %d\n", param[0], cntp);
        param[++cntp] = 0;
        if (fork() == 0) {
            exec(param[0], param);
            // exec() will return only if a error occurs
            exit(1);
        }
        else {
            wait(0);
        }
    }

    exit(0);
}
```



#### 6. make grade

![image-20240408204847240](image-20240408204847240.png)



#### 7. 遗留问题

头文件的引入顺序不对会导致类型未定义等错误，为什么 Java 中没碰到过？两种语言的模块管理怎么对比来看？

> #include 和其他 # 开头的命令（宏定义、条件编译）一样，为预处理命令，在程序编译的第一步预处理（gcc -E）中被执行对应的文本层面的操作，即将对应的头文件内容展开，生成对应的 .i 文件（需要 -o 指定生成文件，否则默认输出到控制台，仍为文本类型可直接查看）。由于展开是按顺序的，如果出现调用先于声明的情况就会出错。
>
> 对于 Java 的 import 只是指定了要引用的类，而具体的被引用类的代码则在执行的时候才调入内存，这个步骤是类加载器来实现的。





