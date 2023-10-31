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

## 终端登录

&emsp;&emsp;太复杂了，暂时没看。

## 网络登录

&emsp;&emsp;同上。

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

-   一个会话可以由一个`控制终端` (control terminal)。通常是终端设备 (终端登录的情况) 或者 伪终端设备 (网络登录的情况)。
-   建立与控制终端连接的会话首进程被称为`控制进程` (control process)。
-   一个会话中的几个进程组可被分成一个`前台进程组` (foreground process group) 以及一个或多个`后台进程组`  (background process group)。
-   如果一个会话有一个控制终端，则它有一个前台进程组，其他进程组为后台进程组。
-   无论何时键入终端的中断或退出键，都会将中断或退出信号发送给前台进程组中的所有进程。
-   如果终端接口检测到调制解调器 (或网络) 断开连接，则将挂断信号发送至控制进程。

![](02.png)

&emsp;&emsp;通常，在登录时，控制终端会被自动建立。

&emsp;&emsp;有时，不管程序的标准输入，标准输出是否被重定向，程序都要与控制终端交互。保证程序能与控制终端对话的方法是 open 文件 /dev/tty。在内核中，此特殊文件是控制终端的同义语。如果程序没有控制终端，则对此设备的 open 将失败。

## 函数 tcgetpgrp, tcsetpgrp 和 tcgetsid

&emsp;&emsp;
