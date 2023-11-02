---
title: APUE 01 - 基础知识
date: 2023-09-17 12:26:13
updated: 2023-09-20 19:00:00
tags:
    - APUE
    - C
categories:
    - APUE
---

&emsp;&emsp;所有操作系统都为他们所运行的程序提供服务。

<!-- more -->

## UNIX 体系结构

&emsp;&emsp;严格意义上讲，`操作系统` (operating system) 是一种软件，它控制计算机硬件资源，提供程序运行环境，这种软件称为`内核` (kernel)。

![](01.png)

&emsp;&emsp;内核的接口被称为`系统调用` (system call)，公共函数库构建在系统调用接口之上。应用程序既可以使用公共函数库，也可以使用系统调用。shell 是一个特殊的应用程序，为运行其他应用程序提供了一个接口。

&emsp;&emsp;广义上讲，操作系统包含了内核和一些其他软件如：系统实用程序 (system utility)，应用程序，shell 以及公共函数库等。Linux 是 GNU 操作系统的内核，这种操作系统称为 GNU/Linux 操作系统，一些人简称为 Linux。

## 登录

1.   登录名

&emsp;&emsp;用户在登录 UNIX 系统时，先键入登录名，然后键入口令。系统在其口令文件 (/etc/passwd) 中查看登录名。口令文件中的登录项由 7 个以冒号分隔的字段组成：登录名 (arthur)，加密口令 (x)，数字用户 ID (1000)，数字组 (1000)，注释字段 (arthur)，起始目录 (/home/arthur)，shell 程序 (/bin/zsh)。

```
arthur:x:1000:1000:arthur:/home/arthur:/bin/zsh
```

&emsp;&emsp;目前，所有的系统已将加密口令移到另一个文件中。

2.   shell

&emsp;&emsp;用户登录后，系统通常会先显示一些系统信息，然后用户就可以向 shell 程序键入命令。shell 是一个命令行解释器，它读取用户输入，然后执行命令。shell 的用户输入通常来自于终端 (交互式 shell)，有时来自于文件 (shell 脚本)。下面是 UNIX 中常见的 shell：

![](02.png)

&emsp;&emsp;系统从口令文件中相应用户登录项的最后一项知道应该为该用户执行哪一个 shell。

&emsp;&emsp;几乎所有的 UNIX 系统都提供 Bourne shell。所有的 BSD 版本都提供 C shell。Korn shell 是 Bourne shell 的继任者，它支持一些 Bourne shell 没有的特色功能。Bourne-again shell 是 GNU shell，它遵循 POSIX 标准，所有的 Linux 系统都提供这种 shell，它支持 C shell 和 Korn shell 两者的特色功能。

## 文件和目录

1.   文件系统

&emsp;&emsp;UNIX 文件系统是目录和文件的一种层次结构，所有的东西的起点都是`根` (root) 目录，根目录的名称是字符`/`。

&emsp;&emsp;`目录` (directory) 是一个包含目录项的文件。在逻辑上，可以认为每个目录项都包含一个文件名，同时还包含说明该文件属性的信息。文件属性指文件类型 (普通文件还是目录等)，文件大小，文件所有者，文件权限，文件最后修改时间等。

>   目录项的逻辑视图与实际存放在磁盘上的方式是不同的。UNIX 文件系统大多数实现并不在目录项中存放属性，这是因为当一个文件具有多个硬链接时，很难保持多个属性副本之间的同步。

2.   文件名

&emsp;&emsp;目录中的各个名字称为`文件名` (filename)。只有斜线 `/`和空字符这两个字符不能出现在文件名中。斜线用来分隔构成呢该路径名的各文件名，空字符用来终止一个路径名。尽管如此，好的习惯还是只使用常用印刷字符的一个子集作为文件名，POSIX.1 推荐只使用：字母，数字，句点，短横线，下划线作为文件名。

&emsp;&emsp;创建新目录时会自动创建两个文件名：`.`和`..`。点指向当前目录，点点指向父目录。在根目录中，点点与点相同。

>   早期的 UNIX 系统中的文件系统限制文件名的最大长度为 14 个字符。而现在几乎所有商业化的 UNIX 文件系统都支持超过 255 个字符的文件名。

3.   路径名

&emsp;&emsp;由斜线分隔的一个或多个文件名组成的序列 (可以以斜线开头) 构成`路径名` (pathname)。以斜线开头的路径名称为`绝对路径`，否则称为`相对路径`。相对路径指向相对于当前目录的文件。文件系统根的名字 (/) 是一个特殊的绝对路径，它不包含文件名。

&emsp;&emsp;使用 ls(1) 命令可以列出一个目录中所有文件的名字，ls(1) 的简单实现：

```c
#include "apue.h"
#include <dirent.h>

int main(int argc, char **argv)
{
    DIR *dp;
    struct dirent *dirp;

    if (argc != 2)
        err_quit("usage: ls <directory name>\n");

    if ((dp = opendir(argv[1])) == NULL)
        err_sys("can't open %s", argv[1]);

    while ((dirp = readdir(dp)) != NULL)
        printf("%s\n", dirp->d_name);

    closedir(dp);
}
```

&emsp;&emsp;ls(1) 是 UNIX 系统的惯用表示法，用以引用 UNIX 系统手册中的一个特定项。ls(1) 引用第一部分中的 ls 项。各部分通常用数字 1 - 8 编号，每个部分中的各项则按字母顺序排列。

&emsp;&emsp;关于这个程序：

-   系统头文件 <dirent.h> 包含了函数 opendir 和 readdir 的原型，以及 dirent 结构的定义。
-   opendir 函数返回指向 DIR 结构的指针，接下来将该指针传给 readdir 函数，该函数会依次读取每个目录项并返回一个指向 dirent 结构的指针，当目录中无目录项可读时返回 NULL。结构 dirent 包含了一些文件的信息，如 d_name 成员就是文件的文件名。

4.   工作目录

&emsp;&emsp;每个进程都有一个`工作目录` (working directory)，也称为`当前工作目录` (current working directory)。所有的相对路径都从工作目录开始解释，进程可以用 chdir 函数更改工作目录。

5.   起始目录

&emsp;&emsp;登录时，工作目录设置为`起始目录` (home directory)，该起始目录从口令文件中相应用户的登录项中取得。

## 输入和输出

1.   文件描述符

&emsp;&emsp;`文件描述符` (file descriptor) 通常是一个小的非负整数，内核用以标识一个特定进程正在访问的文件。当内核打开一个现有文件或者创建一个新文件时，它都返回一个文件描述符。在读写文件时，可以使用这个文件描述符。

2.   标准输入，输出和错误

&emsp;&emsp;按照惯例，每当运行一个新程序时，所有的 shell 都为其打开 3 个文件描述符，即`标准输入`，`标准输出`以及`标准错误`。如果不做特殊处理，则这 3 个描述符都链接向终端。大多数 shell 都提供一种方法，使其中任何一个或者所有描述符都能重新定向到某个文件。

```bash
ls > file.txt
```

&emsp;&emsp;执行 ls 命令，其标准输出重新定向到名为 file.txt 的文件。

3.   不带缓冲的 I/O

&emsp;&emsp;函数 open，read，write，lseek 以及 close 提供了不带缓冲的 I/O，这些函数都使用文件描述符。

&emsp;&emsp;下面的程序会将从标准输入读取的任意内容复制到标准输出：
```c
#include "apue.h"
#include <unistd.h>

#define BUFFSIZE 4096

int main(void)
{
    int n;
    char buf[BUFFSIZE];

    while ((n = read(STDIN_FILENO, buf, BUFFSIZE)) > 0)
        if (write(STDOUT_FILENO, buf, n) != n)
            err_sys("write error");

    if (n < 0)
        err_sys("read error");
}
```

&emsp;&emsp;关于这个程序：

-   头文件 <unistd.h> 和两个常量 STDIN_FILENO 和 STDOUT_FILENO 是 POSIX 标准的一部分。<unistd.h> 包含了很多 UNIX 系统服务的函数原型，如 read 和 write。
-   常量 STDIN_FILENO 和 STDOUT_FILENO 指定了标准输入和标准输出的文件描述符。在 POSIX 标准中，他们分别是 0 和 1，为了可读性，最好使用常量名。

-   函数 read 返回读取的字节数，用以后面的 write 函数，当到达输入文件尾端 EOF 时，read 返回 0。如果发生一个错误，read 返回 -1。

&emsp;&emsp;如果以以下方式运行它：

```bash
./a.out > data
```

&emsp;&emsp;此时，标准输入还是终端，标准输出被重定向到文件 data，标准错误也是终端。如果 data 不存在，则 shell 会创建它。当用户在控制台键入文件结束符 (通常是 Ctrl + D)，将终止本次复制。

&emsp;&emsp;若以下列方式运行该程序：

```bash
./a.out < infile > outfile
```

&emsp;&emsp;会将名为 infile 的文件复制到 outfile 文件中。

4.   标准 I/O

&emsp;&emsp;标准 I/O 函数为哪些不带缓冲的 I/O 函数提供了一个带缓冲的接口。使用标准 I/O 函数无需担心如何选取最佳的缓冲区大小，并且其还简化了对输入行的处理。

&emsp;&emsp;下面的程序类似上一个程序，它将标准输入复制到标准输出：

```c
#include "apue.h"

int main(void)
{
    int c;
    while ((c = getc(stdin)) != EOF)
        if (putc(c, stdout) == EOF)
            err_sys("output error");
    
    if (ferror(stdin))
        err_sys("input error");
}
```

&emsp;&emsp;关于这个程序：

-   函数 getc 定义在头文件 <stdio.h> 中，类似的还有常用的 fgets printf 等。
-   函数 getc 一次读取一个字符，然后函数 putc 将此字符输出到标准输出。读到最后一个字节时，getc 返回常量 EOF。标准 I/O 常量 stdin 和 stdout 分别代表标准输入和标准输出。

## 程序和进程

1.   程序

&emsp;&emsp;`程序` (program) 是一个存储在磁盘上某个目录中的文件。内核使用 exec 函数，将程序读入内存，并执行程序。

2.   进程和进程 ID

&emsp;&emsp;程序的执行示例被称为`进程` (process)。某些操作系统使用`任务` (task) 表示正在被执行的程序。

&emsp;&emsp;UNIX 系统确保每个进程都有一个唯一的数字标识符，称为`进程 ID` (process ID)，进程 ID 总是一个非负整数。

&emsp;&emsp;下面的程序用于打印进程 ID：

```c
#include "apue.h"

int main(void)
{
    printf("hello world from process ID %ld\n", (long)getpid());
}
```

3.   进程控制

&emsp;&emsp;有 3 个用于控制进程的主要函数：fork，exec 和 waitpid (exec 是一类函数，它有 7 个变种)。

&emsp;&emsp;下面的程序实现了一个简单的 shell：

```c
#include "apue.h"
#include <sys/wait.h>

int main()
{
    char buf[MAXLINE];
    pid_t pid;
    int status;

    printf("%% "); /* print prompt */
    while (fgets(buf, MAXLINE, stdin) != NULL) {
        if (buf[strlen(buf) - 1] == '\n')
            buf[strlen(buf) - 1] = 0; /* replace newline with null */

        if ((pid = fork()) < 0)
            err_sys("fork error");
        else if (pid == 0) { /* chile */
            execlp(buf, buf, (char *)0);
            err_ret("couldn't execute: %s", buf);
            exit(127);
        }

        /* parent */
        if ((pid = waitpid(pid, &status, 0)) < 0)
            err_sys("waitpid error");
        printf("%% ");
    }
}
```

&emsp;&emsp;关于这个程序：

-   标准 I/O 函数 fgets 从标准输入中一次读取一行：当读取内容长度小于 MAXLINE (且没有遇到换行符) 时，它会返回这部分内容，并追加一个空字符。当读取的内容长度大于等于 MAXLINE 时，它只会读取 MAXLINE - 1 个字符，因为最后需要一个空字符结尾。如果提前遇到换行符，它会保留换行符并结束读取，然后追加一个空字符。如果读取的第一个字符就是文件结束符 EOF，那么它会返回 NULL。

-   函数 fork 用于创建一个调用进程的副本，称为`子进程`。fork 函数会返回两次，一次在父进程中，返回子进程的进程 ID，一次在子进程中，返回 0。
-   在子进程中调用 execlp 函数执行新程序文件。而父进程希望等待子进程终止，通过调用 waitpid 函数实现，其参数指定要等待进程的 ID，并且它还保存子进程的退出状态 (status 变量)。
-   该程序的主要限制是不能向所执行的命令传递参数，为了传递参数，要先分析输入行，然后用某种约定把参数分开 (如空格或制表符)，再将分隔后的各个参数传递给 execlp 函数。

4.   线程和线程 ID

&emsp;&emsp;通常，一个进程只有一个控制`线程` (thread) - 某一时刻执行的一组机器指令。

&emsp;&emsp;一个进程内的所有线程共享同一地址空间，文件描述符，栈以及与进程相关的属性。因为他们能访问同一存储区，所以各线程在访问共享数据时需要采取同步措施以避免不一致性。

&emsp;&emsp;与进程相同，线程也用 ID 表示。但是，线程 ID 只在它所属的进程内起作用，一个进程中的线程 ID 在另一个进程中没有意义。当一个进程中对某个特定线程进行特殊处理时，可以使用该线程的 ID 引用它。

&emsp;&emsp;控制线程的函数与控制进程类似，但另有一套。

## 出错处理

&emsp;&emsp;当 UNIX 系统函数出错时，通常会返回一个负值，而且整型变量 errno 通常被设置为具有特定信息的值。例如，open 函数如果成功会返回一个非负文件描述符，如果出错则返回 -1。在 open 出错时，有大约 15 种 errno 值。而有些函数如返回指向对象指针的，在出错时会返回一个 NULL 指针而不是负值。

&emsp;&emsp;文件 <errno.h> 中定义了 errno 以及可赋予它的各种常量，这些常量都以字符 E 开头。UNIX 系统手册第 2 部分的第一页，intro(2) 列出所有这些出错常量。

>   在 Linux 中，出错常量在 errno(3) 中列出。

&emsp;&emsp;POSIX 和 ISO C 将 errno 定义为一个符号，它扩展成一个可修改的整型`左值` (lvalue)。它可以是一个包含出错编号的整数，也可以是一个返回出错编号的函数。

```c
/* before */
extern int errno;

/* now in Linux */
extern int *__errno_location(void);
#define errno (*__errno_location());
```

&emsp;&emsp;这是为了支持在多线程环境中，每个线程都可以独立访问他们自己的局部 errno，以避免线程间相互干扰。

&emsp;&emsp;使用 errno 有两条规则：

1.   如果没有出错，其值不会被例程清除。
2.   任何函数都不会将 errno 设置为 0。

&emsp;&emsp;C 标准定义了两个函数，用于打印出错信息：
```c
#include <string.h>
char *strerror(int errnum);

#include <stdio.h>
void perror(const char *msg);
```

&emsp;&emsp;strerror 函数将 errnum 映射为一个出错消息字符串，并返回指向字符串的指针。perror 函数基于 errno 的当前值，在标准错误上产生一条出错消息，然后返回：它首先输出 msg 字符串，然后是一个冒号和空格，接着是对应 errno 的出错信息，最后是一个换行符。

&emsp;&emsp;一个简单的例子：

```c
#include "apue.h"
#include <errno.h>

int main(int argc, char **argv)
{
    fprintf(stderr, "EACCES: %s\n", strerror(EACCES));
    errno = ENOENT;
    perror(argv[0]);
}
```

&emsp;&emsp;可以将 <errno.h> 中定义的各种出错分为两类：致命性的和非致命性的。发生致命性错误时，无法执行恢复动作，最多能做的是在用户屏幕上打印一条出错消息或者是将一条出错消息写入日志文件中，然后退出。对于非致命性错误，有时可以进行较妥善的处理。大多数非致命性错误都是暂时的 (如资源短缺)，对于资源相关的非致命性出错的典型恢复操作是延迟一段时间，然后重试。

## 用户标识

1.   用户 ID

&emsp;&emsp;口令文件登录项中的用户 ID 是一个数值，它向系统标识各个不同的用户。系统在确定一个用户登录名的同时，确定其用户 ID。用户不能更改其用户 ID。通常每个用户有一个唯一的用户 ID，下面是内核如何使用用户 ID 来检验该用户是否有执行某些操作的权限的流程：

&emsp;&emsp;用户 ID 为 0 的用户为`根用户` (root) 或`超级用户` (superuser)。在口令文件中，通常有一个登录项，其登录名为 root，这种用户有超级用户特权。如果一个进程有超级用户特权，则大多数文件权限检查都不再进行。某些操作系统功能只向超级用户提供，超级用户对系统有自由的支配权。

2.   组 ID

&emsp;&emsp;口令文件登录项也包含用户的`组 ID` (group ID)，它是一个数值。组 ID 也是由系统管理员在指定用户登录名时分配的。一般来说，在口令文件中有多个登录项具有相同的组 ID。组 ID 用于将若干用户集合到项目或者部门中去，这种机制允许同组的各个成员之间共享资源。

&emsp;&emsp;组文件将组名映射为数值的组 ID，通常是 /etc/group。

&emsp;&emsp;对于存放在磁盘上的每个文件，文件系统都存储该文件所有者的用户 ID 和组 ID，存储数值 ID 而不是完整的 ASCII 字符串是为了节省空间，和加速比较操作。

>   早期的 UNIX 系统存储用户 ID 和组 ID 都是 2 字节，现代 UNIX 系统大多使用 4 字节。

3.   附属组 ID

&emsp;&emsp;除了在口令文件中对一个登录名指定一个组 ID 外，大多数 UNIX 系统还允许一个用户附属于另外的一些组 (大多数 UNIX 系统至少支持 16 个附属组)。登录时，读文件 /etc/group，寻找列有该用户作为其成员的前 16 个记录项就可以得到该用户的附属组 ID。

## 信号

&emsp;&emsp;`信号` (signal) 用于通知进程发生了某种情况。进程有以下三种处理信号的方式：

1.   忽略信号。有些信号表示硬件异常，例如除以 0 或访问进程地址空间以外的存储单元等。因为这些异常产生的后果不确定，所以不推荐使用这种处理方式。
2.   按系统默认方式处理。对于除零异常，系统默认的处理方式是终止该进程。

3.   提供一个函数，信号发生时调用该函数，这被称为捕捉该信号。

&emsp;&emsp;很多情况都会产生信号，终端键盘上有两种产生信号的方法，分别是`中断键` (interrupt key) (通常是 Delete 键或 Ctrl + C) 和`退出键` (quit key) (通常是 Ctrl + \\)，他们被用于中断当前运行的进程。另一种产生信号的方法是调用 kill 函数，在一个进程中调用 kill 函数可以向另一个进程发送一个信号。但是有限制：向一个进程发送信号时，此进程必须是目的进程的所有者或者是超级用户。

&emsp;&emsp;修改之前的 shell 程序，让其能够捕捉 SIGINT 信号：

```c
#include "apue.h"
#include <sys/wait.h>

static void sig_int(int sig);

int main()
{
    char buf[MAXLINE];
    pid_t pid;
    int status;

    if (signal(SIGINT, sig_int) == SIG_ERR)
        err_sys("signal error");

    printf("%% "); /* print prompt */
    while (fgets(buf, MAXLINE, stdin) != NULL) {
        if (buf[strlen(buf) - 1] == '\n')
            buf[strlen(buf) - 1] = 0; /* replace newline with null */

        if ((pid = fork()) < 0)
            err_sys("fork error");
        else if (pid == 0) { /* child */
            execlp(buf, buf, (char *)0);
            err_ret("couldn't execute: %s", buf);
            exit(127);
        }

        /* parent */
        if ((pid = waitpid(pid, &status, 0)) < 0)
            err_sys("waitpid error");
        printf("%% ");
    }
}

void sig_int(int sig)
{
    printf("interrupt\n%% ");
}
```

## 时间值

&emsp;&emsp;历史上，UNIX 系统使用过两种不同的时间值。

1.   日历时间。该值是自协调世界时 (Coordinated Universal Time, UTC) 1970 年 1 月 1 日 00:00:00 这个特定时间以来所经历的秒数累计值。系统基本数据类型 time_t 用于保存这种时间值。
2.   进程时间。也被称为 CPU 时间，用以度量进程使用的 CPU 资源。进程时间以时钟滴答计算，每秒钟曾取 50 60 或 100 个时钟滴答。系统基本数据类型 clock_t 保存这种时间值。

&emsp;&emsp;当度量一个进程的执行时间时，UNIX 为一个进程维护了 3 个进程时间值：

-   时钟时间。时钟时间又称为`墙上时钟时间` (wall clock time)，它是进程运行的时间总量。
-   用户 CPU 时间。用户时间是执行用户指令所用的时间量。
-   系统 CPU 时间。系统时间是为该进程执行内核程序所经历的时间，如进程执行 read 和 write 所花费的时间就计入该值。

&emsp;&emsp;要取得任一进程的时钟时间，用户时间和系统时间可以使用命令 time(1)：

```bash
cd /usr/include
time -p grep _POSIX_SOURCE */*.h > /dev/null
```

## 系统调用和库函数

&emsp;&emsp;所有操作系统都提供多种服务的入口点，由此程序向内核请求服务。各种版本的 UNIX 实现都提供良好定义，数量有限，直接进入内核的入口点，这些入口点被称为`系统调用` (system call)。系统调用接口总是在 UNIX 用户手册第 2 部分说明，是用 C 语言定义的，与具体系统如何调用一个系统调用的实现技术无关。

&emsp;&emsp;UNIX 所使用的技术是为每个系统调用在标准 C 库中设置一个具有相同名字的函数。用户进程用标准 C 调用序列来调用这些函数，然后，函数又用系统所要求的技术调用相应的内核服务。从应用角度考虑，可以将系统调用设为 C 函数。

&emsp;&emsp;UNIX 用户手册第 3 部分定义了程序员可以使用的通用库函数，虽然这些函数可能会调用一个或多个内核的系统调用，但是他们并不是内核的入口点。

&emsp;&emsp;系统调用和库函数之间有本质区别，如存储空间分配函数 malloc，有多种方法可以实现存储空间分配和回收。UNIX 系统调用中处理存储空间分配的是 sbrk(2)，它不是一个通用的存储器管理器，但是 malloc 函数很有可能使用 sbrk 系统调用。下面的图片展示了他们之间的关系：

![](03.png)

&emsp;&emsp;应用程序既可以调用系统调用也可以调用库函数，很多库函数则会调用系统调用：

![](04.png)

&emsp;&emsp;系统调用和库函数之间的另一个差别是：系统调用通常提供一种最小接口，而库函数通常提供比较复杂的功能。
