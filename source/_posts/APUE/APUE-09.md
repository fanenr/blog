---
title: APUE 09 - 进程关系
date: 2023-10-24 12:36:15
tags:
    - APUE
    - C
categories:
    - APUE
---

&emsp;&emsp;本章将更详细的说明 POSIX 引入的会话概念。

<!-- more -->

## TODO 终端登录

&emsp;&emsp;太复杂了，暂时没看。

## TODO 网络登录

&emsp;&emsp;太复杂了，暂时没看。

## 进程组

&emsp;&emsp;每个进程除了有一个进程 ID 外，还属于一个进程组。进程组是一个或多个进程的集合，通常它们在同一作业中结合起来。同一进程组的各个进程接收来自同一终端的各种信号。每个进程组有一个唯一的进程组 ID，进程组 ID 也是一个正整数，可以用 pid_t 类型存储。

&emsp;&emsp;函数 getpgrp 返回调用进程的进程组 ID：

```c
#include <unistd.h>

pid_t getpgrp(void);

/* Returns: process group ID of calling process */
```

&emsp;&emsp;SUS 定义了函数 getpgid 函数来获取指定进程的进程组 ID：

```c
#include <unistd.h>

pid_t getpgid(pid_t pid);

/* Returns: process group ID if OK, −1 on error */
```

&emsp;&emsp;若 *pid* 为 0，则返回调用进程的进程组 ID。

&emsp;&emsp;每个进程组有一个组长进程，组长进程的进程 ID 等于该进程组的 ID。只要在某个进程组中有一个进程存在，则该进程组就存在，这与其组长是否终止无关。进程组的生命周期是从该进程组被创建开始的，直到进程组内最后一个进程终止结束。

&emsp;&emsp;进程调用 setpgid 可以加入一个现有的进程组或者创建一个新的进程组：

```c
#include <unistd.h>

int setpgid(pid_t pid, pid_t pgid);

/* Returns: 0 if OK, −1 on error */
```

&emsp;&emsp;setpgid 函数将 *pid* 进程的进程组 ID 设置为 *pgid*。如果这两个参数相等，则将 *pid* 进程设置为组长进程。若 *pid* 为 0，则使用调用进程的 PID。若 *pgid* 为 0，则将 *pid* 指定的进程 ID 用作进程组 ID。

&emsp;&emsp;一个进程只能为它自己或其子进程设置进程组 ID。并且，在它的子进程调用 exec 后，它就不能再更改该子进程的进程组 ID 了。

## 会话

&emsp;&emsp;`会话` (session) 是一个或多个进程组的集合，例如：

![](01.png)

&emsp;&emsp;通常由 shell 管道将几个进程编成一组。上面的会话可能是由下面的命令形成的：

```bash
proc1 | proc2 &
proc3 | proc4 | proc5
```

&emsp;&emsp;进程调用 setsid 建立一个新会话：

```c
#include <unistd.h>

pid_t setsid(void);

/* Returns: process group ID if OK, −1 on error */
```

&emsp;&emsp;如果调用进程不是一个进程组的组长，则此函数会创建一个新会话：

1.   调用进程变成新会话的`会话首进程` (session leader)。此时，该进程是新会话中的唯一进程。
2.   该进程成为一个新进程组的组长，新进程组的组 ID 是其 PID。
3.   该进程没有控制终端。如果调用 setsid 之前该进程有一个控制终端，那么这种联系也被切断。

&emsp;&emsp;如果调用进程是一个组长，则函数返回出错。为了保证不出现这种情况，通常先调用 fork，然后使其父进程终止，而子进程则继续。子进程继承了父进程的进程组 ID，而进程 ID 是新分配的，所以二者一定不相等。

&emsp;&emsp;SUS 只说明了会话，没有定义会话的标识：会话 ID。因为会话首进程的 ID 是唯一的，所以其 PID 可用作会话 ID。函数 getsid 返回会话首进程的进程 ID：

```c
#include <unistd.h>

pid_t getsid(pid_t pid);

/* Returns: session leader’s process group ID if OK, −1 on error */
```

&emsp;&emsp;如果 *pid* 为 0，getsid 返回调用进程的会话 ID。但如果 *pid* 不属于调用者所属的会话，则调用进程将无法获取会话 ID。

## 控制终端

&emsp;&emsp;会话和进程组还有一些其他特性：

-   一个会话可以有一个`控制终端` (control terminal)。通常是终端设备 (终端登录的情况) 或伪终端设备 (网络登录的情况)。
-   建立与控制终端连接的会话首进程被称为`控制进程` (control process)。
-   一个会话中的进程组可被分成一个`前台进程组` (foreground process group) 以及一个或多个`后台进程组`  (background process group)。
-   如果一个会话有一个控制终端，则它有一个前台进程组，其他进程组为后台进程组。
-   无论何时键入终端的中断或退出键，都会将中断或退出信号发送给前台进程组中的所有进程。
-   如果终端接口检测到调制解调器 (或网络) 断开连接，则将挂断信号发送至控制进程。

![](02.png)

&emsp;&emsp;通常，在登录时，控制终端会被自动建立。

&emsp;&emsp;有时，不管程序的标准输入，标准输出是否被重定向，程序都要与控制终端交互。保证程序能与控制终端对话的方法是 open 文件 /dev/tty。在内核中，此特殊文件是控制终端的同义语。如果程序没有控制终端，则对此设备的 open 将失败。

## 函数 tcgetpgrp, tcsetpgrp 和 tcgetsid

&emsp;&emsp;需要一种方法来通知内核哪一个进程组是前台进程组，这样，终端设备驱动程序就能直到将终端输入和终端产生的信号发送到何处：

```c
#include <unistd.h>

pid_t tcgetpgrp(int fd);
/* Returns: process group ID of foreground process group if OK, −1 on error */

int tcsetpgrp(int fd, pid_t pgrpid);
/* Returns: 0 if OK, −1 on error */
```

&emsp;&emsp;函数 tcgetpgrp 返回与 *fd* 终端设备关联的前台进程组 ID。

&emsp;&emsp;如果进程有一个控制终端，则该进程可以调用 tcsetpgrp 将前台进程组 ID 设置为 *pgrpid*。并且 *pgrpid* 应当是同一会话中的某个进程组 ID，*fd* 必须是该会话的控制终端。

&emsp;&emsp;大多数应用程序并不直接调用这两个函数，它们通常由作业控制 shell 调用。

&emsp;&emsp;函数 tcgetsid 可以获取会话 ID：

```c
#include <termios.h>

pid_t tcgetsid(int fd);

/* Returns: session leader’s process group ID if OK, −1 on error */
```

&emsp;&emsp;给出控制 TTY 的文件描述符，它返回该终端设备所关联的会话 ID (控制进程的 PID/GID)。

## 作业控制

&emsp;&emsp;作业控制是 BSD 引入的一个特性，它允许在一个终端上启动多个作业 (进程组)，它控制哪一个作业可以访问终端以及哪些作业在后台运行。使用作业控制有下列要求：

1.   支持作业控制的 shell。
2.   内核中的终端驱动程序必须支持作业控制。
3.   内核必须提供对某些作业控制信号的支持。

&emsp;&emsp;从 shell 使用作业控制功能的角度观察，用户可以在前台或后台启动作业。一个作业只是几个进程的集合，通常是一个进程管道：

```bash
vi main.c
pr *.c | lpr &
make all &
```

&emsp;&emsp;当启动一个后台作业时，shell 赋予它一个作业标识符，并在前台打印一个或多个进程 ID：

```bash
$ make all > Make.out &
[1]  1475
$ pr *.c | lpr &
[2]  1490
$ # input enter
[2] + Done  pr *.c | lpr &
[1] + Done  make all > Make.out &
```

&emsp;&emsp;make 的作业编号是 1，启动的进程 ID 是 1475。第二个管道的作业编号是 2，其第一个进程是 1490。当作业完成且键入回车时，shell 会打印作业完成状态。

&emsp;&emsp;用户可以键入 3 个特殊字符使终端驱动程序产生信号，它们会被发送给前台进程组：

1.   中断字符 (Ctrl + C) 产生 SIGINT。
2.   退出字符 (Ctrl + \\) 产生 SIGQUIT。
3.   挂起字符 (Ctrl + Z) 产生 SIGTSTP。

&emsp;&emsp;用户可以有一个前台作业和多个后台作业，通常只有前台作业接收终端输入。如果后台作业试图读终端，这不是一个错误，但终端驱动程序将检测这种情况，并向该后台作业发送一个特定信号 SIGTTIN。该信号通常会停止 (挂起) 此后台作业，而 shell 也会检测这种情况并通知用户。用户可以用 shell 命令将此作业转换为前台作业运行，然后它就可以正常的读终端了。

&emsp;&emsp;默认是允许后台作业写终端的，但用户可以禁止这一行为：

```bash
stty tostop
```

&emsp;&emsp;若用户禁止了后台作业写终端，则当该作业试图写终端时，终端驱动程序会检测到这一行为，并向该作业发送 SIGTTOU 信号，然后该作业会阻塞。读终端一样，用户可以将作业转换为前台作业继续运行。

![](03.png)

## TODO shell 执行程序

&emsp;&emsp;暂时用不上。

## TODO 孤儿进程组

&emsp;&emsp;暂时用不上。

## TODO FreeBSD 实现

&emsp;&emsp;暂时用不上。
