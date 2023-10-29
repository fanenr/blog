---
title: APUE 10
date: 2023-10-27 17:01:57
tags:
    - APUE
    - C
categories:
    - APUE
---

&emsp;&emsp;信号是软件中断，它提供了一种处理异步事件的方法。

<!-- more -->

## 信号概念

&emsp;&emsp;每个信号都有一个名字，这些名字都以 SIG 开头。V7 有 15 种不同的信号，SVR4 和 4.4BSD 均有 31 种不同的信号，OS X 和 Linux 都支持 31 种信号。此外 POSIX 的实时扩展还支持另外的应用程序定义的信号，SUSv4 已经把实时信号接扩扩展移至基础规范说明中了。

&emsp;&emsp;在头文件 <signal.h> 中，信号名都被定义为正整数常量 (信号编号)，不存在编号为 0 的信号。

&emsp;&emsp;很多条件可以产生信号：

-   当用户按下某些终端建时，引发终端产生信号。
-   硬件产生信号：除数为 0，无效的内存引用等。这些条件由硬件检测到，并通知内核。然后内核为该条件发生时正在运行的进程产生适当的信号。
-   进程调用 kill(2) 函数将任意信号发送给另一个进程或进程组。能发送的要求是：接收信号的进程和发送信号的进程的所有者必须相同，或发送进程的所有者是 root 用户。
-   用户可用 kill(1) 命令将信号发送给其他进程，此命令只是 kill 函数的接口。
-   当内核通知一个进程发生了某些事件时，软件条件也可以产生信号。

&emsp;&emsp;信号是异步事件的经典实例，产生信号的事件对于进程来说是随机出现的。进程不能通过简单的测试某个变量来判断是否发生了一个信号，而是必须告诉内核：在某个信号发生时，执行一些操作。这个操作又被称为信号的处理或与信号相关的动作，它包含 3 类：

1.   忽略此信号

&emsp;&emsp;大多数信号都可使用这种方式进行处理，但有两种信号绝不能被忽略：SIGKILL 和 SIGSTOP。原因是：它们向内核或超级用户提供了终止进程的可靠方法，此外，如果忽略了某些由硬件产生的信号，则接下来进程运行的行为将是未定义的。

2.   捕捉信号

&emsp;&emsp;捕捉信号指：通知内核在发生某种信号时，调用一个用户函数。通过这种方式，用户可以针对某些事件做一些特殊的处理。但是，不能捕捉 SIGKILL 和 SIGSTOP 信号。

3.   执行系统默认动作

&emsp;&emsp;对于大多数信号而言，系统的默认动作是终止该进程，下面是每一种信号的默认动作：

![](01.png)

&emsp;&emsp;终止 + core 表示终止时在进程的当前工作目录生成一个 core 文件，core 文件中复制了该进程的内存映像和寄存器状态，可以使用调试工具 (gdb) 进一步分析进程终止的原因。

&emsp;&emsp;有 5 种情况不生成 core 文件：

1.   进程是设置用户 ID 的，而且当前用户并非可执行文件的所有者。
2.   进程是设置组 ID 的，而且当前用户并非可执行文件的组所有者。
3.   用户没有当前工作目录的写权限。
4.   文件已存在，而且用户对该文件没有写权限。
5.   文件太大。

## 函数 signal

&emsp;&emsp;UNIX 系统信号机制最简单的接口是 signal 函数：

```c
#include <signal.h>

void (*signal(int signo, void (*func)(int)))(int);

/* Returns: previous disposition of signal (see following) if OK, SIG_ERR on error */
```

&emsp;&emsp;signal 的函数签名比较复杂，可以简化如下：

```c
typedef void Sigfun(int);
Sigfunc *signal(int signo, Sigfunc *func);
```

&emsp;&emsp;参数 *signo* 是之前表中的信号名，*func* 可以是用户自定义函数地址或两个常量之一：SIG_IGN，SIG_DFL。如果指定 SIG_IGN，则向内核表示忽略此信号。如果指定 SIG_DFL，则表示接到此信号后的动作是系统默认动作。当指定自定义函数地址时，内核会在信号发生时调用该函数，这种处理被称为`捕捉信号`，此函数则被称为`信号处理程序` (signal handler) 或`信号捕捉函数` (signal-catching function)。

&emsp;&emsp;系统头文件 <signal.h> 中可能会包含以下定义：

```c
#define SIG_ERR  (void (*)())-1
#define SIG_DFL  (void (*)())0
#define SIG_IGN  (void (*)())1
```

&emsp;&emsp;signal 函数的返回值是指向之前信号处理程序的指针。

1.   程序启动

&emsp;&emsp;当使用 exec 函数执行一个新程序时，因为进程的正文段会被替换，所以之前设置的所有信号捕捉函数都将被重置为默认动作。但是，之前设置忽略的行为不会改变。

&emsp;&emsp;例如，在交互式 shell 中运行后台程序：

```bash
cc main.c &
```

&emsp;&emsp;shell 会将后台进程对中断和退出信号的处理方式设置为忽略。于是，按下中断字符时就不会影响到后台进程，如果不这么处理，当按下中断字符时，它不仅会终止前台进程，也会终止所有后台进程。

2.   进程创建

&emsp;&emsp;当一个进程调用 fork 时，其子进程继承父进程的信号处理方式。因为子进程在开始时复制了父进程的内存映像，所以信号处理函数地址在子进程中是有意义的。

## 不可靠的信号

&emsp;&emsp;在早期的 UNIX 版本中 (如 V7)，信号是不可靠的 -- 信号可能会丢失：一个信号发生了，但进程可能一直不知道。同时，进程对信号的控制能力也很差：进程只能捕捉信号或忽略它，但是不能阻塞信号。

&emsp;&emsp;早期版本中的一个问题是，进程每次收到一个信号并对其进行处理后，该信号随即就被重置为默认动作。程序不得不再次设置它：

```c
int sig_int(); /* custom signal handler */

main()
{
    signal(SIGINT, sigint); /* establish handler */    
}

sig_int()
{
    signal(SIGINT, sigint); /* reestablish handler */
    /* ... */
}
```

&emsp;&emsp;这种早期的处理方式有一个问题：在信号发生之后到信号处理程序调用 signal 之间有一个时间窗口。在这段时间内，可能发生另一次中断信号，第二个信号会造成执行默认动作，而中断的默认动作是终止进程。这段程序在大多数情况下都能正常运行，但是实际上是有问题的。

&emsp;&emsp;早期版本中的另一个问题是，进程不能阻塞信号：通知内核不忽略某个信号，并在其发生时记住它，然后在进程准备好时再通知它。基于早期 UNIX 的程序只能这么做：捕捉一个信号，然后设置一个代表信号已经发生的标志：

```c
int sig_int();
int sig_int_flag;

main()
{
    signal(SIGINT, sigint);
    while (sig_int_flag == 0)
        pause();
}

sig_int()
{
    signal(SIGINT, sig_int);
    sig_int_flag = 1;
}
```

&emsp;&emsp;进程调用 pause 函数进入休眠，直到捕捉到一个信号。当捕捉到信号时，信号处理程序将标志 sig_int_flag 设置为 1。从信号处理程序返回时，内核将唤醒进程，进程会检测到标志非 0，然后接着做一些其他的事情。这里不仅有上一个问题存在，还存在另一个问题：如果信号在测试标志之后，调用 pause 之前发生，并且信号以后再也不会发生了，那么进程可能会陷入永久休眠。

## 中断的系统调用

&emsp;&emsp;早期 UNIX 系统的一个特性是：如果进程在执行一个低速系统调用的阻塞期间捕捉到一个信号，则该系统调用就被中断不再继续执行，并且该系统调用将返回出错，errno 被设置为 EINTR。

>   要区分函数和系统调用，当捕捉到某个信号时，被中断的是内核中执行的系统调用。

&emsp;&emsp;为了支持这种特性，系统调用被分成两类：低速系统调用和其他系统调用。低速系统调用是可能会使进程永远阻塞的一类系统调用，包括：

-   如果某些类型文件 (如管道，终端设备和网络设备) 的数据不存在，读操作可能使调用者永远阻塞。
-   如果这些数据不能被相同类型文件立即接收，写操作可能会使调用者永远阻塞。
-   在某种条件发生之前打开某些类型的文件，可能会发生阻塞。
-   函数 pause 和 wait。
-   某些 ioctl 操作。
-   某些进程间通信操作。

&emsp;&emsp;由于系统调用会被捕捉的信号中断，所以必须显式的处理其出错返回：

```c
again:
    if ((n = read(fd, buf, BUFFSIZE)) < 0) {
        if (errno == EITNR)
            goto again;
        /* handle other errors */
    }
```

&emsp;&emsp;为了帮助程序不必处理被中断的系统调用，4.2BSD 引入了某些被中断系统调用的自动重启动，它们包括：ioctl, read, write, readv, writev, wait 和 waitpid。前 5 个函数只有对低速设备进行操作时才会被信号中断，而 wait 和 waitpid 在捕捉到信号时总是被中断。

&emsp;&emsp;不同的 UNIX 实现可能对处理信号中断系统调用的行为不同：

![](02.png)

## 可重入函数

&emsp;&emsp;当进程捕捉到信号时，其正在执行的正常指令序列会被信号处理程序临时中断，它会首先执行该信号处理程序中的指令。如果信号处理程序正常返回，则继续执行进程被中断时的指令序列。但在信号处理程序中，无法判断捕捉到信号时程序执行到何处，如果进程正在执行 malloc，而此时信号处理程序又调用 malloc，那么 malloc 为进程维护的堆内存块链表就可能被覆盖。

&emsp;&emsp;SUS 说明了在信号处理程序中保证调用安全的函数，这些函数是可重入和`异步信号安全` (async-signal safe) 的。除了可重入外，在信号处理操作期间，内核也会阻塞任何会引起不一致的信号发送。

![](03.png)

&emsp;&emsp;没有列入上表的函数大多数是不可重入的，因为：

-   它么使用静态数据结构。
-   它们调用 malloc 或 free。
-   它们是标准 IO 函数，标准 IO 库的很多函数都使用全局数据结构。

&emsp;&emsp;此外，由于每个线程只有一个 errno 变量，所以信号处理程序可能会修改其原先值：如果 main 程序刚设置 errno 就被中断，而信号处理程序中又调用了可能设置 errno 的函数，当正常返回时，main 将获得被替代的 errno 值。所以，信号处理程序在调用这类函数时应当先保存 errno，并在调用结束后恢复。

## 可靠信号术语和语义

&emsp;&emsp;信号术语`产生` (generation)：当导致信号产生的事件发生时，内核会为进程产生一个信号。事件可以是硬件异常，软件条件，终端产生的信号或调用 kill 函数。内核产生信号的主要动作是：在进程表的 pending 结构中设置一个与该信号关联的位。

&emsp;&emsp;当进程状态发生变化时 (由内核态变为用户态)，内核会结合 pending 和阻塞信号集 blocked 判断要向进程通知哪些信号 (pending & \~blocked)，通知动作即内核向进程`递送` (delivery) 了一个信号。在信号产生和递送之间的时间间隔内，信号处于`未决`状态 (pending)。

&emsp;&emsp;进程可以选择阻塞信号递送。如果进程阻塞了某个信号，并且对该信号的动作是系统默认动作或捕捉，则内核会保持进程对该信号的未决状态。直到进程解除对此信号的阻塞，或者将此信号的动作设置为忽略。内核在递送一个被阻塞的信号给进程时 (而非信号产生时)，才决定对它的处理方式。

&emsp;&emsp;注意，当产生信号的事件发生时，内核设置 pending 是无条件的。

&emsp;&emsp;如果在进程解除对某个信号的阻塞之前，该信号产生了很多次，那么解除阻塞后，POSIX 允许系统递送该信号一次或多次。如果递送了多次，则称这些信号进行了排队。除非支持 POSIX 实时扩展，否则大多数 UNXI 并不对信号排队，而是只递送该信号一次。

&emsp;&emsp;如果有多个信号要递送给一个进程，POSIX 并未规定这些信号的递送顺序。但是，POSIX 基础部分建议：优先递送与进程当前状态相关的信号，如 SIGSEGV。

&emsp;&emsp;每个进程都有一个`信号屏蔽字` (signal mask)，它规定了内核要阻塞递送到当前进程的信号集。该信号屏蔽字中每一位都对应一种可能的信号，如果某个信号对应的位被设置，则它当前就是被阻塞的。

&emsp;&emsp;信号编号可能超过一个整型所包含的二进制位数，因此 POSIX 定义了一个新的数据类型 sigset_t，它可以容纳一个信号集。

## 函数 kill 和 raise

&emsp;&emsp;kill 函数将信号发送给进程或进程组，raise 函数则允许进程向自身发送信号：

```c
#include <signal.h>

int kill(pid_t pid, int signo);
int raise(int signo);

/* Both return: 0 if OK, −1 on error */
```

&emsp;&emsp;raise 其实是 kill 和 getpid 的组合：

```c
kill(getpid(), signo)
```

&emsp;&emsp;kill 的 *pid* 参数有 4 种不同的情况:

| pid       | meaning                                               |
| --------- | ----------------------------------------------------- |
| pid > 0   | 将信号发送给 PID 为 *pid* 的进程。                    |
| pid == 0  | 将信号发送给与发送进程同进程组的所有进程 (包括自身)。 |
| pid < 0   | 将信号发送给进程组 ID 为 *pid* 中的所有进程。         |
| pid == -1 | 将信号发送给进程有权限给它们发送信号的所有进程。      |

&emsp;&emsp;调用 kill 给其他进程发送信号需要一定的权限，上面所指的发送对象都在此限制下：

-   特权级进程可以向任一进程发送信号。
-   init 进程是特例，它只接收它要捕捉的信号 (防止被杀死)。
-   如果发送者的 ruid 或 euid 等于接收者的 ruid 或 suid：

![](04.png)

-   SIGCONT 信号需要特殊处理：进程总是能发送该信号给同一会话的其他任一进程。

&emsp;&emsp;POSIX 将 0 编码为空信号，如果 *signo* 是 0，则 kill 仍执行正常的错误检查，但不发送信号。这常被用来确定一个特定进程是否仍然存在，因为向一个并不存在的进程发送信号会返回 -1，并设置 errno 为 ESRCH。但应注意：UNIX 系统在经过一段时间后会复用进程 ID。

## 函数 alarm 和 pause

&emsp;&emsp;使用 alarm 函数可以设置一个定时器，在将来的某个时刻该定时器会超时。当定时器超时时，将为调用进程产生 SIGALRM 信号，如果忽略或不捕捉此信号，则其默认动作是终止调用进程：

```c
#include <unistd.h>

unsigned int alarm(unsigned int seconds);

/* Returns: 0 or number of seconds until previously set alarm */
```

&emsp;&emsp;参数 *seconds* 是产生 SIGALRM 信号需要经过的秒数，当这一时刻到达时，内核将产生该信号，由于进程调度的延迟，所以进程从得到控制到能够处理该信号还需要一个时间间隔。

&emsp;&emsp;每个进程只能有一个闹钟时间，如果调用 alarm 时，之前为该进程注册的闹钟时间还未超时，则该闹钟时间的余留值将作为本次 alarm 函数调用的返回值返回。以前注册的闹钟时间则被新值取代。

&emsp;&emsp;如果进程之前设置了一个闹钟，那么传递值 为 0 的 *seconds* 可以清除该闹钟。如果之前闹钟还未超时，则 alarm 会返回剩余的秒数。

&emsp;&emsp;pause 函数使调用进程挂起直至捕捉到一个信号：

```c
#include <unistd.h>

int pause(void);

/* Returns: −1 with errno set to EINTR */
```

&emsp;&emsp;只有进程执行了一个信号处理程序并从其返回时，pause 才返回。在这种情况下，pause 返回 -1，并且 errno 被设置为 EINTR。

## 信号集

&emsp;&emsp;不同的信号的编码可能超过一个整型量所包含的位数，因此不能使用整型量中的一位代表一种信号：即信号集不能使用一个整型来表示。为此，POSIX 定义了新数据类型 sigset_t 以包含一个信号集，并且定义了下面 5 种处理信号集的函数：

```c
#include <signal.h>

int sigemptyset(sigset_t *set);
int sigfillset(sigset_t *set);
int sigaddset(sigset_t *set, int signo);
int sigdelset(sigset_t *set, int signo);
/* All four return: 0 if OK, −1 on error */

int sigismember(const sigset_t *set, int signo);
/* Returns: 1 if true, 0 if false, −1 on error */
```

&emsp;&emsp;函数 sigemptyset 初始化 *set* 指向的信号集，清除其中所有的信号。函数 sigfillset 初始化 *set* 指向的信号集，但是它使其包含所有的信号。在使用信号集之前，一定要调用二者之一来初始化。

&emsp;&emsp;一旦已经初始化信号集，就可以在其中增，删特定的信号。函数 sigaddset 将一个信号加入信号集中，函数 sigdelset 从一个信号集中删除某个信号。函数 sigmember 用来测试一个信号集是否包含某个特定的信号。

## 函数 sigprocmask

&emsp;&emsp;调用 sigprocmask 函数可以检测或修改进程的信号屏蔽字：

```c
#include <signal.h>

int sigprocmask(int how, const sigset_t *restrict set, sigset_t *restrict oset);

/* Returns: 0 if OK, −1 on error */
```

&emsp;&emsp;如果 *oset* 是非空指针，那么其将保存进程当前的信号屏蔽字。

&emsp;&emsp;如果 *set* 是非空指针，则函数根据 *how* 的值来决定如何修改进程信号屏蔽字：

![](05.png)

&emsp;&emsp;如果 *set* 是空指针，则 *how* 值就无意义。在调用 sigprocmask 后如果有任何未决的，不再阻塞的信号，则在 sigprocmask 返回之前至少将其中之一递送给进程。

## 函数 sigpending

&emsp;&emsp;sigpending 函数可以获取进程中当前处于未决状态的信号集：

```c
#include <signal.h>

int sigpending(sigset_t *set);

/* Returns: 0 if OK, −1 on error */
```

&emsp;&emsp;具体来说，sigpending 会返回进程表中的 pending 集合。

## 函数 sigaction

&emsp;&emsp;sigaction 函数的功能是检查或修改与指定信号关联的处理动作，它可以取代 signal 函数：

```c
#include <signal.h>

int sigaction(int signo, const struct sigaction *restrict act,
              struct sigaction *restrict oact);

/* Returns: 0 if OK, −1 on error */
```

&emsp;&emsp;其中，*signo* 是要处理的信号编号。若 *act* 是非空指针，则执行修改动作。若 *oact* 是非空指针，则函数会在其指定位置存储该信号之前的动作。

```c
struct sigaction
{
    void (*sa_handler)(int); /* addr of signal handler, */
                             /* or SIG_IGN, or SIG_DFL */
    sigset_t sa_mask;        /* additional signals to block */
    int sa_flags;            /* signal options, Figure 10.16 */
    void (*sa_sigaction)(int, siginfo_t *, void *); /* alternate handler */
};
```

&emsp;&emsp;当更改信号动作时，字段 sa_handler 包含信号的处理函数地址。如果其指向一个用户自定义函数 (不是 SIG_IGN 和 SIG_DFL)，则内核在调用该捕捉函数前会将 sa_mask 信号集加入信号屏蔽字中，并在信号处理程序返回时恢复原来的信号屏蔽字。这样，在调用信号处理程序时就能阻塞某些信号。

&emsp;&emsp;在信号处理程序被调用时，内核会也会将当前信号临时加入屏蔽字中。如果在其调用期间，信号产生了多次，那么结束调用时内核才会解除阻塞，并且通常也只递送一次。

&emsp;&emsp;*act* 结构中的 sa_flags 字段指定对信号处理的各个选项：

![](06.png)

&emsp;&emsp;若使用了上面的 SA_SIGINFO 标志，内核会在递送信号时调用函数 sa_sigaction 而不是 sa_handler，前者可以获取更多信息：

```c
void handler(int signo, siginfo_t *info, void *context);
```

&emsp;&emsp;siginfo_t 结构包含了与信号产生原因的有关信息，POSIX 要求该结构至少包含 si_signo 和 si_code 成员，符合 XSI 的实现至少包含以下字段：

```c
struct siginfo
{
    int si_signo;          /* signal number */
    int si_errno;          /* if nonzero, errno value from errno.h */
    int si_code;           /* additional info (depends on signal) */
    pid_t si_pid;          /* sending process ID */
    uid_t si_uid;          /* sending process real user ID */
    void *si_addr;         /* address that caused the fault */
    int si_status;         /* exit value or signal number */
    union sigval si_value; /* application-specific value */
    /* possibly other fields also */
};

union sigval si_value {
    int sigval_int;
    void *sigval_ptr;
};
```

&emsp;&emsp;应用程序在递送信号时，可以在 si_value.sigval_int 中传递一个整型或在 si_value.sigval_ptr 中传递一个通用指针值。

&emsp;&emsp;当递送某些信号时，si_code 会包含信号发生的原因：

![](07.png)

&emsp;&emsp;若信号是 SIGCHLD，则设置 si_pid, si_status 和 si_uid 字段。若信号是 SIGBUS, SIGILL, SIGFPE 或 SIGSEGV，则 si_addr 包含造成故障的根源地址，si_errno 包含错误编码。

&emsp;&emsp;信号处理程序的 *context* 参数是通用指针，它可被强转为 ucontext_t 结构类型，该结构标识信号传递时进程的上下文，该结构至少包含以下字段：

```c
ucontext_t *uc_link;    /* pointer to context resumed when */
                        /* this context returns */
sigset_t uc_sigmask;    /* signals blocked when this context */
                        /* is active */
stack_t uc_stack;       /* stack used by this context */
mcontext_t uc_mcontext; /* machine-specific representation of */
                        /* saved context */
```

&emsp;&emsp;uc_stack 字段描述了当前上下文使用的栈，至少包含以下成员：

```c
void *ss_sp;    /* stack base or pointer */
size_t ss_size; /* stack size */
int ss_flags;   /* flags */
```

-----

&emsp;&emsp;很多平台都使用 sigaction 函数实现 signal：

```c
#include "apue.h"

/* Reliable version of signal(), using POSIX sigaction(). */
Sigfunc *signal(int signo, Sigfunc *func)
{
    struct sigaction act, oact;

    act.sa_handler = func;
    sigemptyset(&act.sa_mask);
    act.sa_flags = 0;
    if (signo == SIGALRM) {
#ifdef SA_INTERRUPT
        act.sa_flags |= SA_INTERRUPT;
#endif
    } else {
        act.sa_flags |= SA_RESTART;
    }
    if (sigaction(signo, &act, &oact) < 0)
        return (SIG_ERR);
    return (oact.sa_handler);
}
```

&emsp;&emsp;注意：必须用 sigemptyset 函数初始化 act 结构中的 sa_mask 成员。该实现中对除了 SIGALRM 外的信号都设置了 SA_RESTART 标志，希望内核自动重启被中断的系统调用。SUS 的 XSI 扩展规定，除非说明 SA_RESTART 标志，否则 sigaction 函数不再自动重启被中断的系统调用。有些系统定义了 SA_INTERRUPT 标志，其意义与 SA_RESTART 正好相反。

&emsp;&emsp;下面是另一个实现，它阻止系统自动重启被中断的系统调用：

```c
#include "apue.h"

Sigfunc *signal_intr(int signo, Sigfunc *func)
{
    struct sigaction act, oact;
    act.sa_handler = func;
    sigemptyset(&act.sa_mask);
    act.sa_flags = 0;
#ifdef SA_INTERRUPT
    act.sa_flags |= SA_INTERRUPT;
#endif
    if (sigaction(signo, &act, &oact) < 0)
        return (SIG_ERR);
    return (oact.sa_handler);
}
```

## 函数 sigsetjmp 和 siglongjmp

&emsp;&emsp;
