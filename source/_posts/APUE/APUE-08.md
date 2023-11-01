---
title: APUE 08 - 进程控制
date: 2023-10-16 19:00:28
tags:
    - APUE
    - C
categories:
    - APUE
---

&emsp;&emsp;本章描述 UNIX 系统的进程控制，包括创建新进程，执行程序和进程终止。

<!-- more -->

## 进程标识

&emsp;&emsp;每个进程都有一个用非负整型表示的唯一的的进程 ID。

&emsp;&emsp;虽然进程 ID 是唯一的，但是进程 ID 是可以复用的。当一个进程终止后，其进程 ID 就成为复用的候选者。大多数 UNIX 系统实现延迟复用算法，使得赋予新进程的 ID 不同于最近终止进程所使用的 ID。这防止了将新进程误认为是之前的以终止的进程。

&emsp;&emsp;系统中有一些专用进程：ID 为 0 的进程通常是调度进程，常常被称为`交换进程` (swapper)。该进程是内核的一部分，它并不执行磁盘上的任何程序，因此也被称为系统进程。ID 为 1 的进程通常是 init 进程，在自举过程结束时由内核调用，该进程的程序文件通常是 /sbin/init。此进程负责在自举内核后启动一个 UNIX 系统，init 通常读取与系统有关的初始化文件 (/etc/rc* 文件或 /etc/inittab 文件，以及目录 /etc/init.d 中的文件)，并将系统引导到一个状态 (如多用户)。init 进程绝不会终止。它是一个普通的用户进程 (不同于 swapper，它不是内核中的进程)，但它以超级用户权限运行。

&emsp;&emsp;除了进程 ID，每个进程还有一些其他标识符，用下列函数获取：

```c
#include <unistd.h>

pid_t getpid(void);
/* Returns: process ID of calling process */

pid_t getppid(void);
/* Returns: parent process ID of calling process */

uid_t getuid(void);
/* Returns: real user ID of calling process */

uid_t geteuid(void);
/* Returns: effective user ID of calling process */

gid_t getgid(void);
/* Returns: real group ID of calling process */

gid_t getegid(void);
/* Returns: effective group ID of calling process */
```

&emsp;&emsp;注意，这些函数都没有出错返回。

## 函数 fork

&emsp;&emsp;一个现有进程可以调用 fork 函数创建一个新进程：

```c
#include <unistd.h>

pid_t fork(void);

/* Returns: 0 in child, process ID of child in parent, −1 on error */
```

&emsp;&emsp;由 fork 函数创建的新进程称为`子进程` (child process)。fork 函数被调用一次，但返回两次：在子进程中的返回值是 0，在父进程中的返回值是子进程的进程 ID。

&emsp;&emsp;子进程和父进程继续执行 fork 调用之后的指令。子进程是父进程的副本：子进程获得父进程数据空间，堆和栈的副本。副本即指他们将不共享这些内存区域，但是他们共享正文段。

&emsp;&emsp;由于 fork 之后常跟着 exec，所以很多实现并不直接复制父进程数据段，栈和堆的完全副本。而是使用`写实复制`(Copy On Write, COW) 技术。这些区域由父进程和子进程共享，内核将他们的访问权限设置为只读。如果父进程或子进程试图修改这些区域，那么内核将为被修改区域的那一页制作一个副本。

>   Linux 提供了另一种新进程创建函数 clone(2) 系统调用。这是一种 fork 的推广形式，它允许调用者控制哪些部分由父进程和子进程共享。

&emsp;&emsp;下面的例子演示了 fork 的效果：

```c
#include "apue.h"

int char globvar = 6; /* external variable in initialized data */
buf[] = "a write to stdout\n";

int main(void)
{
    int var;
    pid_t pid; /* automatic variable on the stack */
    var = 88;
    if (write(STDOUT_FILENO, buf, sizeof(buf) - 1) != sizeof(buf) - 1)
        err_sys("write error");
    printf("before fork\n"); /* we don’t flush stdout */
    if ((pid = fork()) < 0) {
        err_sys("fork error");
    } else if (pid == 0) { /* child */
        globvar++;         /* modify variables */
        var++;             /* parent */
    } else {
        sleep(2);
    }
    printf("pid = %ld, glob = %d, var = %d\n", (long)getpid(), globvar, var);
    exit(0);
}
```

&emsp;&emsp;运行结果：

```bash
$ ./a.out
a write to stdout
before fork
pid = 430, glob = 7, var = 89  # child’s variables were changed
pid = 429, glob = 6, var = 88  # parent’s copy was not changed

$ ./a.out > temp.out
$ cat temp.out
a write to stdout
before fork
pid = 432, glob = 7, var = 89
before fork
pid = 431, glob = 6, var = 88
```

-----

&emsp;&emsp;注意：在 fork 之后，是父进程先执行还是子进程先执行是不确定的，这取决于内核所使用的调度算法。如果要求父进程和子进程之间相互同步，则需使用某种形式的进程间通信。

&emsp;&emsp;一个有意思的地方是：分别以终端交互方式和重定向文件方式运行该程序，*before fork* 字符串被写入的次数不同。这是因为：当以交互方式运行该程序时，STDOUT_FILENO 关联终端，所以其标准 I/O 流是行缓冲的，所以在 fork 之前调用 printf 就会立即冲洗缓冲区。但是，以重定向文件方式运行该程序时，其标准 I/O 流是全缓冲的，所以在 printf 之后，缓冲区并没有立即冲洗。当 fork 之后，子进程将获得父进程中一些内存区域的副本，标准 I/O 缓冲区就在其中。所以，当 fork 之后，带有未冲洗数据的缓冲区也被复制到子进程了。最终，在 exit 被调用时，*before fork* 就会被两个进程分别写入。

&emsp;&emsp;文件共享

&emsp;&emsp;如果父进程的标准输出被重定向了，那么新创建的子进程的标准输出也会被重定向。这是因为 fork 会将父进程打开的所有文件描述符都复制到子进程，这种复制类似 dup 函数。

&emsp;&emsp;调用 fork 之后，父进程和子进程将会共享同一文件表表项，这其中就包括了文件偏移量：

![](01.png)

&emsp;&emsp;如果父子进程写入同一描述符指向的文件，但又没有任何形式的同步，那么它们的输出就会相互混合。

&emsp;&emsp;除了打开文件之外，子进程还会继承父进程的以下属性：

-   实际用户 ID，实际组 ID，有效用户 ID，有效组 ID。
-   附属组 ID。
-   进程组 ID。
-   会话 ID。
-   控制终端。
-   设置用户 ID 标志和设置组 ID 标志。
-   当前工作目录。
-   根目录。
-   文件模式创建屏蔽字。
-   信号屏蔽和安排。
-   文件描述符标志 close-on-exec。
-   环境。
-   链接的共享存储段。
-   存储映像。
-   资源限制。

&emsp;&emsp;父进程和子进程之间的区别如下：

-   fork 的返回值不同。
-   进程 ID 不同。
-   它们各自的父进程 ID 不同。
-   子进程的 tms_utime, tms_stime, tms_cutime 和 tms_ustime 的值被设置为 0。
-   子进程不继承父进程设置的文件锁。
-   子进程的未处理闹钟将被清除。
-   子进程的未处理信号集设置为空集。

&emsp;&emsp;fork 有以下两种用法：

1.   一个父进程希望复制自己，使父进程和子进程同时执行不同的代码段，这在网络服务器中很常见。
2.   一个进程要执行一个不同的程序，这是 shell 中常见的。

&emsp;&emsp;某些操作系统将第 2 种用法中的两个操作 (fork 和 exec) 组合成一个操作，称为 spawn。

## 函数 vfork

&emsp;&emsp;函数 vfork 的调用序列和返回值与 fork 相同，但二者的语义不同。

>   vfork 起源于 BSD，但是后来被废弃。

&emsp;&emsp;vfork 用于创建一个新进程，而新进程的目的是 exec 一个程序。vfork 和 fork 的区别在于：vfork 不将父进程的地址空间完全复制到子进程中，在子进程调用 exec 或 exit 之前，它在父进程的空间中运行。这种工作方式在某些 UNIX 系统实现中提高了效率，但如果子进程修改数据，进行函数调用，或者没有调用 exec 或 exit 都会带来位置后果。

&emsp;&emsp;vfork 和 fork 的另一个区别是：vfork 保证子进程先运行，在它调用 exec 或者 exit 之后父进程才可能被调度运行。如果子进程在调用这两个函数之前而依赖父进程的进一步动作，就会导致死锁。

&emsp;&emsp;例子：

```c
#include "apue.h"

int globvar = 6;

int main(void)
{
    int var;
    pid_t pid;
    /* external variable in initialized data */
    /* automatic variable on the stack */
    var = 88;
    printf("before vfork\n");
    /* we don’t flush stdio */
    if ((pid = vfork()) < 0) {
        err_sys("vfork error");
    } else if (pid == 0) {
        /* child */
        globvar++;
        /* modify parent’s variables */
        var++;
        _exit(0);
        /* child terminates */
    }
    /* parent continues here */
    printf("pid = %ld, glob = %d, var = %d\n", (long)getpid(), globvar, var);
    exit(0);
}
```

&emsp;&emsp;运行结果：

```bash
$ ./a.out
before vfork
pid = 29039, glob = 7, var = 89
```

&emsp;&emsp;可以看到：子进程修改了父进程中的 globvar，这是因为 vfork 创建的子进程运行在父进程地址空间下。并且，子进程调用了 _exit 而不是 exit，这是为了以防 exit 会关闭流底层的文件描述符。

>   大多数 exit 的现代实现都不会关闭流底层的文件描述符了，因为在进程终止时，内核会自动关闭。

## 函数 exit

&emsp;&emsp;进程有 5 种正常终止方式和 3 种异常终止方式：

1.   在 main 函数中调用 return 语句 (等效于调用 exit 函数)。
2.   调用 exit 函数。
3.   调用 _exit 或 _Exit 函数。
4.   进程的最后一个线程在其启动例程中执行 return 语句。
5.   进程的最后一个线程调用 pthread_exit 函数。

-----

1.   调用 abort 函数，它产生 SIGABRT 信号。
2.   当进程收到某些信号时。
3.   最后一个线程对`取消` (cancellation) 请求作出响应。

&emsp;&emsp;不管进程如何终止，最后都会执行内核中的同一段代码：这段代码为相应进程关闭所有打开的文件，释放它使用的所有内存等。

&emsp;&emsp;子进程终止时的状态称为终止状态，父进程可以获取其终止状态 (一个数值) 来判断子进程是如何终止的。终止状态有两类：当进程异常终止时，内核 (不是进程本身) 产生一个指示其异常终止原因的终止状态。当进程正常终止时，进程会有一个退出状态，内核会将退出状态转换为终止状态。进程可以将退出状态作为参数传递给 3 个 exit 函数，或者是给 return 语句。正常终止中的第 4 和 5 种情况下，进程的退出状态都是 0。

&emsp;&emsp;如果父进程在子进程之前终止，那么子进程的父进程将变为 init 进程，即它们被 init 进程收养。

&emsp;&emsp;如果子进程在父进程之前终止，内核并不会立即清除子进程的所有痕迹，内核会为子进程保留一定量的信息 (至少包括进程 ID，进程的终止状态以及进程使用的 CPU 时间总量)，父进程可以调用 wait 或 waitpid 函数得到这些信息。

&emsp;&emsp;在 UNIX 中，一个已经终止，但是其父进程尚未对其进行善后处理 (获得终止子进程的有关信息，释放它仍占用的资源) 的进程被称为`僵死进程` (zombie)。命令 ps(1) 将僵死进程的状态打印为 Z。

&emsp;&emsp;如果父进程先于子进程终止，那么子进程将被 init 进程收养，init 进程会保证获取该子进程的终止状态，这就防止了该子进程变成僵死进程。

## 函数 wait 和 waitpid

&emsp;&emsp;当一个进程正常或异常终止时，内核就向其父进程发送 SIGCHLD 信号。因为子进程终止是一个异步事件，所以这种信号也是内核向父进程发送的异步通知。父进程可以选择忽略该信号 (默认行为)，或者提供一个信号处理程序，做一些特定工作。

&emsp;&emsp;若进程调用了 wait 或 waitpid 函数：

-   如果其所有子进程都还在运行，则阻塞。
-   如果一个子进程已终止，正等待父进程获取其终止状态，则取得终止状态并立即返回。
-   如果它没有子进程，则立即出错返回。

&emsp;&emsp;如果进程在收到 SIGCHLD 信号后调用 wait，则期望 wait 能立即返回。但如果在任意时间点调用 wait，则进程可能会阻塞。

```c
#include <sys/wait.h>

pid_t wait(int *statloc);
pid_t waitpid(pid_t pid, int *statloc, int options);

/* Both return: process ID if OK, 0 (see later), or −1 on error */
```

&emsp;&emsp;两个 wait 函数的区别：

-   在一个子进程终止前调用 wait 将使调用者阻塞，而 waitpid 有一个选项可使调用者不阻塞。
-   waitpid 并不等待在其调用后的第一个终止子进程，它有若干选项，可以控制所等待的进程。

&emsp;&emsp;这两个函数的参数 *statloc* 都是一个整型指针，如果 *statloc* 不是空指针，则终止进程的终止状态就会放在该位置中。如果不关心终止状态，则传递空指针。

&emsp;&emsp;进程的终止状态字是由实现定义的，其中某些位表示退出状态 (正常返回)，其他位则指示异常编号 (异常返回)，有 1 位指示是否产生了 core 文件等。POSIX 规定，使用 <sys/wait.h> 中的 4 个宏来判断进程终止的原因，基于这 4 个宏返回的真假，可以进一步使用其他宏取得信息：

![](02.png)

&emsp;&emsp;下面的 pr_exit 使用如上宏打印进程终止类型和状态：

```c
#include "apue.h"
#include <sys/wait.h>

void pr_exit(int status)
{
    if (WIFEXITED(status))
        printf("normal termination, exit status = %d\n", WEXITSTATUS(status));
    else if (WIFSIGNALED(status))
        printf("abnormal termination, signal number = %d%s\n", WTERMSIG(status),
#ifdef WCOREDUMP
               WCOREDUMP(status) ? " (core file generated)" : "");
#else
               "");
#endif
    else if (WIFSTOPPED(status))
        printf("child stopped, signal number = %d\n", WSTOPSIG(status));
}
```

&emsp;&emsp;使用 pr_exit 演示终止状态的各种值：

```c
#include "apue.h"
#include <sys/wait.h>
int main(void)
{
    pid_t pid;
    int status;

    if ((pid = fork()) < 0)   /* child */
        err_sys("fork error");
    else if (pid == 0)
        exit(7);
    if (wait(&status) != pid) /* wait for child */
        err_sys("wait error");
    pr_exit(status);          /* and print its status */

    if ((pid = fork()) < 0)   /* child */
        err_sys("fork error");
    else if (pid == 0)
        abort();              /* generates SIGABRT */
    if (wait(&status) != pid) /* wait for child */
        err_sys("wait error");
    pr_exit(status);          /* and print its status */

    if ((pid = fork()) < 0)   /* child */
        err_sys("fork error");
    else if (pid == 0)
        status /= 0;          /* divide by 0 generates SIGFPE */
    if (wait(&status) != pid) /* wait for child */
        err_sys("wait error");
    pr_exit(status);          /* and print its status */
}
```

&emsp;&emsp;运行结果：

```bash
$ ./a.out
normal termination, exit status = 7
abnormal termination, signal number = 6 (core file generated)
abnormal termination, signal number = 8 (core file generated)
```

&emsp;&emsp;如果有多个子进程，并且要等待指定子进程终止时，早期 UNIX 版本中的做法是，循环调用 wait 直到等待到制定子进程。POSIX 定义了 waitpid 函数来简化这一操作。

&emsp;&emsp;waitpid 中的参数 *pid* 的解释：

| pid       | meaning                                            |
| --------- | -------------------------------------------------- |
| pid == -1 | 等待任意一个子进程。等效于 wait 函数。             |
| pid > 0   | 等待进程 ID 为 pid 的子进程。                      |
| pid == 0  | 等待组 ID 与调用进程的进程组 ID 相同的任一子进程。 |
| pid < -1  | 等待组 ID 为 pid 绝对值的任一子进程。              |

&emsp;&emsp;waitpid 也返回终止子进程的进程 ID，并将该子进程的终止状态存放在 *statloc* 指向的位置。参数 *options* 可以进一步控制 waitpid 的操作，其值或为 0，或为下列常量的按为或结果：

![](03.png)

-----

&emsp;&emsp;如果一个进程 fork 一个子进程，但不要等待子进程终止，也不希望在父进程终止后成为僵死进程，一个实现技巧是：调用 fork 两次。(nb)

```c
#include "apue.h"
#include <sys/wait.h>
int main(void)
{
    pid_t pid;
    if ((pid = fork()) < 0) {
        err_sys("fork error");
    } else if (pid == 0) {
        /* first child */
        if ((pid = fork()) < 0)
            err_sys("fork error");
        else if (pid > 0)
            exit(0);
        /* parent from second fork == first child */
        /*
         * We’re the second child; our parent becomes init as soon
         * as our real parent calls exit() in the statement above.
         * Here’s where we’d continue executing, knowing that when
         * we’re done, init will reap our status.
         */
        sleep(2);
        printf("second child, parent pid = %ld\n", (long)getppid());
        exit(0);
    }
    if (waitpid(pid, NULL, 0) != pid)
        err_sys("waitpid error");
    /* wait for first child */
    /*
     * We’re the parent (the original process); we continue executing,
     * knowing that we’re not the parent of the second child.
     */
}
```

&emsp;&emsp;为了保证第二个子进程打印父进程 ID 时，第一个子进程已经终止，其调用了 sleep 函数。这是因为无法确保 fork 之后，父进程一定先于子进程被调度。

```bash
$ ./a.out
second child, parent pid = 1
```

## 函数 waitid

&emsp;&emsp;waittid 是 SUS 包含的另一个取得进程终止状态的函数，它类似 waitpid，但提供了更多的灵活性。

```c
#include <sys/wait.h>

int waitid(idtype_t idtype, id_t id, siginfo_t *infop, int options);

/* Returns: 0 if OK, −1 on error */
```

&emsp;&emsp;waittid 也允许进程指定要等待的子进程，但是它通过两个单独的参数表示要等待的子进程所属的类型。参数 *id* 的作用与 *idtype* 的值相关，*idtype* 类型如下：

![](04.png)

&emsp;&emsp;参数 *options* 是下列标志的按位或结果：

![](05.png)

&emsp;&emsp;以上标志指示调用者关注哪些状态变化，并且：WCONTINUED, WEXITED 或 WSTOPPED 这 3 个中的之一必须在 *options* 中出现。

&emsp;&emsp;*infop* 参数是指向 siginfo 结构的指针。该结构包含了造成子进程状态改变有关信号的详细信息。

## 函数 wait3 和 wait4

&emsp;&emsp;大多数 UNIX 系统实现提供了 wait3 和 wait4，这两个函数是从 BSD 分支沿袭下来的。它们提供的功能比 wait, waitpid, waittid 所提供的功能多一个，这与附加参数有关。该参数允许内核返回由终止进程及其所有子进程使用的资源情况。

```c
#include <sys/types.h>
#include <sys/wait.h>
#include <sys/time.h>
#include <sys/resource.h>

pid_t wait3(int *statloc, int options, struct rusage *rusage);
pid_t wait4(pid_t pid, int *statloc, int options, struct rusage *rusage);

/*　Both return: process ID if OK, 0, or −1 on error　*/
```

&emsp;&emsp;资源统计信息包括用户 CPU 时间总量，系统 CPU 时间总量，缺页次数，接收到信号的次数等。

![](06.png)

##　竞争条件

&emsp;&emsp;当多个进程都试图对共享数据进行某种处理，而最后的结果又取决于进程的运行顺序时，就认为发生了竞争条件 (race condition)。如果在 fork 之后的某种逻辑显式或隐式的依赖于在 fork 之后是父进程先运行还是子进程先运行，那么 fork 函数将很容易导致竞争条件的产生。

## 函数 exec

&emsp;&emsp;使用 fork 创建一个子进程后，子进程往往会调用一种 exec 函数以执行另一个程序。子进程调用 exec 函数时，该进程执行的程序完全替换为新程序，而新程序则从其 main 函数开始执行。调用 exec 并不创建新进程，所以前后的进程 ID 并未改变。exec 只是用磁盘上的一个程序替换了当前进程的正文段，数据段，堆和栈。

&emsp;&emsp;exec 是一个函数族，它有 7 种变体：

```c
#include <unistd.h>

int execl(const char *pathname, const char *arg0, ... /* (char *)0 */ );
int execv(const char *pathname, char *const argv[]);
int execle(const char *pathname, const char *arg0, .../* (char *)0, char *const envp[] */);
int execve(const char *pathname, char *const argv[], char *const envp[]);
int execlp(const char *filename, const char *arg0, ... /* (char *)0 */ );
int execvp(const char *filename, char *const argv[]);
int fexecve(int fd, char *const argv[], char *const envp[]);

/* All seven return: −1 on error, no return on success */
```

&emsp;&emsp;前 4 个函数都以路径名作为参数，后两个以文件名作为参数，最后一个取文件描述符作为参数。

&emsp;&emsp;指定 *filename* 参数时：

-   如果 *filename* 包含 / 则将其视为路径名。
-   否则就按照 PATH 环境变量，在其指定的各目录搜索可执行文件。

&emsp;&emsp;PATH　环境变量包含了一张目录表，目录之间以 : 分隔，例如：

```
PATH=/bin:/usr/bin:/usr/local/bin/:.
```

&emsp;&emsp;最后的路径前缀 . 表示当前目录 (不建议包含当前目录)。

&emsp;&emsp;如果 execlp 或 execvp 找到了一个可执行文件，但是该文件不是由链接器产生的可执行文件，则认为该文件是一个 shell 脚本，于是尝试调用 /bin/sh 并以该 *filename* 作为 shell 的输入。

&emsp;&emsp;fexecve 函数避免了寻找正确的可执行文件，而是依赖调用进程中文件描述符。调用进程可以使用文件描述符验证所需的文件并且无竞争的执行该文件。

&emsp;&emsp;它们之间的第二个区别与参数表的传递有关 (l 代表 list，v 代表 vector)。函数 execl, execlp 和 execle 要求将新程序的每个命令行参数都说明为一个单独的参数，这种参数表以一个 NULL 指针结尾。其他 4 种函数都接收一个字符串数组的地址。

&emsp;&emsp;注意：对于 execl 系列函数时，最后一个参数应该传递空指针，而不是 0，因为后者实际被解释为一个整型参数，如果 int 的长度与 char　*　不同，那么　exec 的实际参数将出错。

&emsp;&emsp;最后一个区别与向新程序传递环境表相关，以 e 结尾的 3 个函数：execle，execve，fexecve 可以传递一个指向环境变量字符串数组的指针。其他 4 个函数则使用调用进程种的 environ 变量为程序复制现有的环境。通常，一个进程的环境可以被传播给其子进程。

&emsp;&emsp;这 7 个函数之间的区别：

![](07.png)

&emsp;&emsp;每个系统对于参数表和环境表都有一个限制，此限制由 ARG_MAX 给出。为了摆脱对参数长度的限制，可以使用 xargs(1) 命令将长参数表断开成几个部分。例如：

```bash
find /usr/share/man -type f -print  |　xargs grep getrlimit
```

&emsp;&emsp;执行 exec 后进程 ID 不会改变，新程序继承了调用进程的以下属性：

-   进程 ID 和父进程 ID。
-   实际用户 ID 和实际组 ID。
-   附属组 ID。
-   进程组 ID。
-   会话 ID。
-   控制终端。
-   闹钟尚余留的时间。
-   当前工作目录。
-   根目录。
-   文件模式创建屏蔽字。
-   文件锁。
-   进程信号屏蔽。
-   未处理信号。
-   资源限制。
-   nice 值。
-   tms_time, tms_stime, tms_cutime 以及 tms_cstime 值。

&emsp;&emsp;对打开文件的处理与每个文件描述符的 close-on-exec 标志有关，若设置了此标志，则在执行 exec 时关闭该描述符，否则该描述符仍然打开。系统默认行为是保持打开，可以使用 fcntl 关闭该标志。

&emsp;&emsp;POSIX 规定，函数 opendir 会为一个目录流设置 close-on-exec 标志，所以执行 exec 时，所有打开的目录流也会默认被关闭。

&emsp;&emsp;注意，在 exec 前后进程的实际用户 ID 和实际组 ID 不改变，而有效 ID 和有效组 ID 是否改变则取决于可执行文件的设置用户 ID 位和设置组 ID 位是否设置。

&emsp;&emsp;在很多 UNIX 实现种，这 7 个函数种只有 execve 是内核的系统调用，另外 6 个只是库函数，它们进行一些中间处理，而最终都要调用 execve：

![](08.png)

&emsp;&emsp;库函数 execlp 和 execvp 使用 PATH 变量查找第一个包含 *filename* 的可执行文件。库函数 fexecve 使用 /proc 把文件描述符参数转化为路径名，然后调用 execve。

-----

&emsp;&emsp;下面的程序演示了 exec 函数：

```c
#include "apue.h"
#include <sys/wait.h>

char *env_init[] = {"USER=unknown", "PATH=/tmp", NULL};

int main(void)
{
    pid_t pid;

    if ((pid = fork()) < 0) {
        err_sys("fork error");
    } else if (pid == 0) { /* specify pathname, specify environment */
        if (execle("/home/sar/bin/echoall", "echoall", "myarg1", "MY ARG2",
                   (char *)0, env_init) < 0)
            err_sys("execle error");
    }
    if (waitpid(pid, NULL, 0) < 0)
        err_sys("wait error");
    if ((pid = fork()) < 0) {
        err_sys("fork error");
    } else if (pid == 0) { /* specify filename, inherit environment */
        if (execlp("echoall", "echoall", "only 1 arg", (char *)0) < 0)
            err_sys("execlp error");
    }
    exit(0);
}
```

&emsp;&emsp;echoall 程序只是简单的将命令行参数和环境表打印出来：

```c
#include "apue.h"

int main(int argc, char *argv[])
{
    int i;
    char **ptr;
    extern char **environ;
    for (i = 0; i < argc; i++)
        /* echo all command-line args */
        printf("argv[%d]: %s\n", i, argv[i]);
    for (ptr = environ; *ptr != 0; ptr++)
        printf("%s\n", *ptr);
    /* and all env strings */
    exit(0);
}
```

&emsp;&emsp;程序运行结果：

```bash
$ ./a.out
argv[0]: echoall
argv[1]: myarg1
argv[2]: MY ARG2
USER=unknown
PATH=/tmp
$　argv[0]: echoall
argv[1]: only 1 arg
USER=sar
LOGNAME=sar
SHELL=/bin/bash  # 47 more lines that aren’t shown
HOME=/home/sar
```

&emsp;&emsp;第 7 行中的 shell 提示符出现在 argv[0] 之前，这是因为父进程并没有等待子进程终止。

## 更改用户 ID 和组 ID

&emsp;&emsp;在 UNIX 系统中，特权和访问控制是基于用户 ID 和组 ID 的。当程序需要增加特权，或需要访问当前并不允许访问的资源时，就需要更改自己的用户 ID 和组 ID，使得新的 ID 具有合适的特权或访问权限。类似的，当程序需要降低特权或阻止对某些资源的访问时，也需要更换用户 ID 和组 ID，使得新的 ID 不具备相应的特权或访问资源的能力。

&emsp;&emsp;一般而言，在设计应用时，总是试图使用`最小特权` (least privilege) 模型。依照此模型，程序应当只具有为完成给定任务所需的最小特权，这降低了由恶意用户试图欺骗程序而使用特权造成的安全风险。

&emsp;&emsp;可以使用 setuid 函数设置实际用户 ID 和有效用户 ID。类似的，使用 setgid 函数可以设置实际组 ID 和有效组 ID：

```c
#include <unistd.h>

int setuid(uid_t uid);
int setgid(gid_t gid);

/* Both return: 0 if OK, −1 on error */
```

&emsp;&emsp;更改 ID 对调用者是有要求的：

1.   若进程具有超级用户特权 (有效用户 ID 为 0)，则 setuid 函数将进程的实际用户 ID，有效用户 ID以及保存的设置用户 ID 都设置为 *uid*。
2.   若进程没有超级用户特权，但是 *uid* 等于实际用户 ID 或保存的设置用户 ID，则 setuid 只将有效用户 ID 设置为 *uid*。不更改实际用户 ID 和保存的设置用户 ID。
3.   如果上面两个条件都不满足，则 errno 被设置为 EPERM，并返回 -1。

&emsp;&emsp;关于内核为每个进程维护的 3 个用户 ID：

1.   只有超级用户进程可以更改实际用户 ID。通常，实际用户 ID 是由 login(1) 程序设置的，而且绝不会更改它。因为 login(1) 是一个超级用户进程，当它调用 setuid 时，会将进程的 3 个用户 ID 都设置为登录用户的 uid。
2.   仅当可执行文件的设置用户 ID 位被设置时，exec 才会设置有效用户 ID 位。
3.   保存的设置用户 ID 是由 exec 复制有效用户 ID 得到的。如果可执行文件设置了设置用户 ID 位，那么 exec 更改有效用户 ID 后就会保存这个副本。

![](09.png)

&emsp;&emsp;如果一个程序设置了 set-user-ID-root 位，那么执行该程序时，其 euid 和 suid 都会变成 root。如果该程序想调用 setuid 降权，那么进程的 ruid, euid, suid 都会更改，降权之后程序再也没办法恢复 root 权限。一个可行的办法是：使用 seteuid 函数，该函数只修改进程的 euid，而不会改变 suid。

```c
#include <unistd.h>

int seteuid(uid_t uid);
int setegid(gid_t gid);

/* Both return: 0 if OK, −1 on error */
```

&emsp;&emsp;一个非特权进程可以使用 seteuid 将其 euid 修改为其 ruid 或 suid。

&emsp;&emsp;另一组函数是 setreuid，它可以同时更改进程的 euid 和 ruid。但对于非特权进程来说，进程只可以交换 euid 和 ruid 的值，只有特权进程才能随意更改：

```c
#include <unistd.h>

int setreuid(uid_t ruid, uid_t euid);
int setregid(gid_t rgid, gid_t egid);

/* Both return: 0 if OK, −1 on error */
```

![](10.png)

&emsp;&emsp;修改组 ID 的检查逻辑和修改用户 ID 是一致的。同时，附属组 ID 不受 setgid，setregid 和 setegid 的影响。

## 解释器文件

&emsp;&emsp;现今所有的 UNIX 系统都支持解释器文件 (interpreter file)。这种文件是文本文件，其起始行的形式：

```bash
#! pathname [optional-argument]
```

&emsp;&emsp;感叹号和 pathname 之间的空格是可选的，例如：

```bash
#!/bin/bash
```

&emsp;&emsp;pathname 通常是绝对路径名，对它不使用 PATH 进行路径搜索。对这种文件的识别是由内核系统调用 exec 处理的，exec 函数并不执行该文本文件，而是执行 pathname 程序。

&emsp;&emsp;一个例子：

```c
#include "apue.h"
#include <sys/wait.h>

int main(void)
{
    pid_t pid;
    if ((pid = fork()) < 0) {
        err_sys("fork error");
    } else if (pid == 0) {
        /* child */
        if (execl("/home/sar/bin/testinterp", "testinterp", "myarg1", "MY ARG2",
                  (char *)0) < 0)
            err_sys("execl error");
    }
    if (waitpid(pid, NULL, 0) < 0) /* parent */
        err_sys("waitpid error");
    exit(0);
}
```

&emsp;&emsp;执行该程序：

```bash
$ cat /home/sar/bin/testinterp
#!/home/sar/bin/echoarg foo
$ ./a.out
argv[0]: /home/sar/bin/echoarg
argv[1]: foo
argv[2]: /home/sar/bin/testinterp
argv[3]: myarg1
argv[4]: MY ARG2
```

&emsp;&emsp;解释器程序 /home/sar/bin/echoarg 回显每一个命令行参数，当内核 exec 解释器时，argv[0] 是该解释器的 pathname，随后是解释器文件中第一行的 [optional-argument]，最后才是 exec 函数调用传递给它的命令行参数。

&emsp;&emsp;解释器文件仍需是可执行文件才能被 exec。

## 函数 system

&emsp;&emsp;ISO C 定义了 system 函数，其可以很方便的在程序中执行一个命令。但是 system 函数对操作系统的依赖性很强，POSIX 扩展了 system 接口，描述了其在 POSIX 中的行为：

```c
#include <stdlib.h>

int system(const char *cmdstring);

/* Returns: (see below) */
```

&emsp;&emsp;如果 *cmdstring* 是一个空指针，则仅当命令处理程序可用时，system 返回非 0 值，这可以用来测试在一个给定的操作系统环境中是否支持 system 函数 (UNIX 中总是可用)。

&emsp;&emsp;因为 system 中实现了 fork，exec 和 waitpid，因此有 3 种返回值：

1.   fork 失败或 waitpid 返回除 EINTR 之外的出错，system 返回 -1 并设置 errno。
2.   如果 exec 失败 (表示不能执行 shell)，则其返回值如果 shell 执行了 exit(127)。
3.   否则所有 3 个函数都执行成功，那么 system 的返回值是 shell 的终止状态。

&emsp;&emsp;下面是一种对 system 的实现，但它没有对信号进行处理：

```c
#include <errno.h>
#include <unistd.h>
#include <sys/wait.h>

int system(const char *cmdstring)
{
    pid_t pid;
    int status;
    /* version without signal handling */
    if (cmdstring == NULL)
        return (1);
    /* always a command processor with UNIX */
    if ((pid = fork()) < 0) {
        status = -1;
        /* probably out of processes */
    } else if (pid == 0) {
        /* child */
        execl("/bin/sh", "sh", "-c", cmdstring, (char *)0);
        _exit(127);
        /* execl error */
    } else {
        /* parent */
        while (waitpid(pid, &status, 0) < 0) {
            if (errno != EINTR) {
                status = -1; /* error other than EINTR from waitpid() */
                break;
            }
        }
    }
    return (status);
}
```

&emsp;&emsp;shell 的 -c 选项告诉 shell 取下一个命令行参数 (*cmdstring*) 作为命令输入，而不是从标准输入或一个文件中读命令。shell 对以 NULL 字节终止的命令字符串进行语法分析，将它们分成命令行参数。传递给 shell 的实际命令可以包含任一有效 shell 命令。

&emsp;&emsp;注意：用于 exec 的子进程调用 _exit 而不是 exit 终止，这是为了防止任一标准 I/O 缓冲 (这些缓冲会在 fork 时被拷贝到子进程) 在子进程中被冲洗。

&emsp;&emsp;使用 system 而不是直接使用 fork 和 exec 的原因是：system 函数进行了所需的各种出错处理以及各种信号处理。

-----

&emsp;&emsp;绝不应该在一个 set-user-ID-root 的程序中使用 system 函数，这会带来巨大的安全隐患。这是因为：一个设置了 root 设置用户 ID 位的程序被执行时将获得 root 权限，system 函数创建子进程时也会继承其 euid，子进程也将获得 root 特权。

## 进程会计

&emsp;&emsp;大多数 UNIX 系统提供了一个选项以进行进程会计 (process accounting) 处理，启用该选项后，每个进程结束时内核就会写一个会计记录。典型的会计记录包含总量较小的二进制数据，一般包括命令名，所使用的 CPU 时间总量，用户 ID 和组 ID，启动时间等。

&emsp;&emsp;函数 acct 启用和禁用进程会计。唯一使用这个函数的是 accton(8) 命令。超级用户执行一个带路径名参数的 accton 命令启用会计记录。会计记录会写到指定文件中：Linux 是 /var/account/pacct。

&emsp;&emsp;会计记录结构定义在 <sys/acct.h> 中，基本信息如下：

```c
#include <sys/types.h>

typedef u_short comp_t; /* 3-bit base 8 exponent; 13-bit fraction */
struct acct
{
    char ac_flag;    /* flag (see below) */
    char ac_stat;    /* termination status (signal & core flag only) */
                     /* (Solaris only) */
    uid_t ac_uid;    /* real user ID */
    gid_t ac_gid;    /* real group ID */
    dev_t ac_tty;    /* controlling terminal */
    time_t ac_btime; /* starting calendar time */
    comp_t ac_utime; /* user CPU time */
    comp_t ac_stime; /* system CPU time */
    comp_t ac_etime; /* elapsed time */
    comp_t ac_mem;   /* average memory usage */
    comp_t ac_io;    /* bytes transferred (by read and write) */
                     /* "blocks" on BSD systems */
    comp_t ac_rw;    /* blocks read or written */
                     /* (not present on BSD systems) */
    char ac_comm[8]; /* command name: [8] for Solaris, */
    /* [10] for Mac OS X, [16] for FreeBSD, and [17] for Linux */
};
```

&emsp;&emsp;在大多数平台上，时间是以时钟滴答数记录的。成员 ac_flag 记录了进程执行期间的某些事件：

![](11.png)

&emsp;&emsp;会计记录的各个数据都由内核保存在进程表中：在一个新进程被创建时初始化，在进程终止时写一个记录。这产生两个后果：

1.   不能获取永不终止的进程的会计记录如 init。
2.   在会计文件中记录的顺序对应于进程终止的终止顺序，而不是它们的启动顺序。要确定启动顺序则必须读完整个会计记录文件，但是启动时间是以秒记录的，所以无法精确判断启动顺序。

&emsp;&emsp;会计记录对应进程而不是程序，这意味着：如果一个子进程启动了多个程序 (A exec B, B exec C, C exit)，最后只会写一个记录。

## 用户标识

&emsp;&emsp;进程可以很容易的获取其实际用户 ID 和有效用户 ID。如果希望找到运行该程序用户的登录名，一种简单地做法是调用 getpwuid(getuid())，但是如果一个用户有多个登录名，这些登录名对应同一个用户 ID，则应该使用 getlogin 函数获得此登录名：

```c
#include <unistd.h>

char *getlogin(void);

/* Returns: pointer to string giving login name if OK, NULL on error */
```

&emsp;&emsp;如果调用此函数的进程没有连接到用户登录时所用的终端，则函数会失败。

## 进程调度

&emsp;&emsp;UNIX 系统历史上对进程提供的只是基于调度优先级的粗粒度的控制。调度策略和调度优先级是由内核确定的，进程可以通过调整 nice 值选择以更低优先级运行，只有特权进程允许提高调度权限。

&emsp;&emsp;POSIX 实时扩展增加了在多个调度类别中选择的接口，以允许进程进一步细调行为。POSIX 的 XSI 扩展中包含了用于调整 nice 值的接口。

&emsp;&emsp;SUS 中 nice 值的范围在 0 ~ (2\*NZERO)-1 之间，有些实现支持 0 ~ 2\*NZERO。nice 值越小，优先级越高，NZERO 是进程默认的 nice 值。

&emsp;&emsp;进程可以通过 nice 函数获取或更改它的 nice 值，使用这个函数，进程只能影响自己的 nice 值，而不能影响其他任何进程的 nice 值：

```c
#include <unistd.h>

int nice(int incr);

/* Returns: new nice value − NZERO if OK, −1 on error */
```

&emsp;&emsp;参数 *incr* 被增加到调用进程的 nice 值上。如果 *incr* 太大，系统会直接把它降低到最大合法值，并且不给出提示。类似的，如果 *incr* 太小，系统也会直接把它提高到最小合法值。由于 -1 既是合法的返回值也是出错返回值，所以要确定 nice 是否调用成功必须同时检查 errno。

&emsp;&emsp;getpriority 函数可以像 nice 函数一样用于获取进程的 nice 值，但是 getpriority 还可以获取一组相关进程的 nice 值：

```c
#include <sys/resource.h>

int setpriority(int which, id_t who, int value);

/* Returns: 0 if OK, −1 on error */
```

&emsp;&emsp;参数 *which* 可以取 3 个值：PRIO_PROCESS 表示进程，PRIO_PGRP 表示进程组，PRIO_USER 表示实际用户 ID。参数 *which* 控制 *who* 是如何被解释的，*who* 参数选择感兴趣的一个或多个进程。如果 *who* 为 0，则表示调用进程，进程组或用户。如果 *which* 参数作用于多个进程，则返回所有作用进程中优先级最高的 nice 值。

&emsp;&emsp;setpriority 函数可用于为进程，进程组和属于特定用户的所有进程设置优先级：

```c
#include <sys/resource.h>

int setpriority(int which, id_t who, int value);

/* Returns: 0 if OK, −1 on error */
```

&emsp;&emsp;参数 *which* 和参数 *who* 与 getpriority 中的含义一致，参数 *value* 将被增加到 NZERO 上，然后变为新的 nice 值。

## 进程时间

&emsp;&emsp;任一进程都可以调用 times 函数，以获取度量调用进程以及已终止子进程时间的 3 种值：墙上时钟时间，用户 CPU 时间以及系统 CPU 时间：

```c
#include <sys/times.h>

clock_t times(struct tms *buf );

/* Returns: elapsed wall clock time in clock ticks if OK, −1 on error */
```

&emsp;&emsp;此函数填写由 *buf* 指向的 tms 结构，该结构定义如下：

```c
struct tms {
    clock_t tms_utime; /* user CPU time */
    clock_t tms_stime; /* system CPU time */
    clock_t tms_cutime; /* user CPU time, terminated children */
    clock_t tms_cstime; /* system CPU time, terminated children */
};
```

&emsp;&emsp;注意，此结构不包含墙上时钟时间，因为 times 会将墙上时钟时间作为函数返回值返回。针对子进程的两个时间是进程调用 wait 函数族已等待到的时间。

&emsp;&emsp;clock_t 类型值可用 _SC_CLK_TCK (由 sysconf 函数返回的每秒钟时钟滴答数) 转换成秒数。

>   大多数 UNIX 实现都提供了 getrusage(2) 函数，该函数返回 CPU 时间以及指示资源使用情况的另外 14 个值。
