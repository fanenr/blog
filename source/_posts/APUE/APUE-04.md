---
title: APUE 04 - 文件和目录
date: 2023-09-23 20:58:57
updated: 2023-09-26 19:00:00
tags:
    - APUE
    - C
categories:
    - APUE
---

&emsp;&emsp;本章描述文件系统的其他特征和文件的性质。

<!-- more -->

## 函数 stat, fstat, fstatat 和 lstat

&emsp;&emsp;本章主要讨论 4 个 stat 函数以及它们的返回信息。

```c
#include <sys/stat.h>

int stat(const char *restrict pathname, struct stat *restrict buf);
int fstat(int fd, struct stat *buf);
int lstat(const char *restrict pathname, struct stat *restrict buf);
int fstatat(int fd, const char *restrict pathname, struct stat *restrict buf, int flag);

/* All four return: 0 if OK, −1 on error */
```

&emsp;&emsp;一旦给出 *pathname*，stat 函数将返回与此命名文件有关的信息结构。fstat 函数返回在描述符 *fd* 上打开的文件的有关信息。lstat 函数类似 stat，但是当命名文件是一个符号链接时，lstat 返回该符号链接的有关信息，而不是该符号链接引用的文件的信息。

&emsp;&emsp;在 fstatat 函数中，如果 *pathname* 是一个绝对路径，那么 *fd* 将被忽略，*pathname* 就是查询文件的路径名。如果 *pathname* 是一个相对路径，并且 *fd* 是常量 AT_FDCWD，那么就在当前工作目录计算文件的路径名。否则，就以 *fd* 关联的打开目录和 *pathname* 计算文件的路径名。在这两种情况下，根据 *flag* 的取值 (AT_SYMLINK_FOLLOW 或 AT_SYMLINK_NOFOLLOW) 决定是否跟随符号链接，跟随符号链接则返回该符号链接引用的文件的信息，不跟随则返回符号链接本身的信息。

&emsp;&emsp;调用方必须提供一个 *buf* 指针，函数将会填充这个结构体，stat 结构体的定义：

```c
struct stat {
    mode_t st_mode;          /* file type & mode (permissions) */
    ino_t st_ino;            /* i-node number (serial number) */
    dev_t st_dev;            /* device number (file system) */
    dev_t st_rdev;           /* device number for special files */
    nlink_t st_nlink;        /* number of links */
    uid_t st_uid;            /* user ID of owner */
    gid_t st_gid;            /* group ID of owner */
    off_t st_size;           /* size in bytes, for regular files */
    struct timespec st_atim; /* time of last access */
    struct timespec st_mtim; /* time of last modification */
    struct timespec st_ctim; /* time of last file status change */
    blksize_t st_blksize;    /* best I/O block size */
    blkcnt_t st_blocks;      /* number of disk blocks allocated */
};
```

&emsp;&emsp;tiemspec 结构类型按照秒和纳秒定义了时间，在 Linux 下的定义如下：

```c
struct timespec {
    time_t tv_sec; /* typedef long time_t */
    long tv_nsec;
}
```

&emsp;&emsp;stat 结构中的成员大多是基本数据类型，使用 stat 函数最多的地方可能是 ls -l 命令。

## 文件类型

&emsp;&emsp;UNIX 系统的大多数文件是普通文件或目录，但是也有另外一些文件类型：

1.   普通文件

&emsp;&emsp;这是最常用的文件类型，这种文件包含了某种形式的数据。对于 UNIX 而言，数据是文本还是二进制格式并无区别。对普通文件的解释应该由处理该文件的应用程序进行。

>   对于二进制可执行文件，为了执行程序，内核必须理解其格式。所有二进制可执行文件都遵循一种标准化的格式，这种格式使内核能够确定程序文本和数据的加载位置。

2.   目录文件

&emsp;&emsp;这种文件包含了其他文件的名字以及指向与这些文件有关信息的指针。对一个目录文件具有读权限的任一进程都可以读该目录的内容，但是只有内核可以直接写目录文件。进程必须使用系统调用才能更改目录。

3.   块特殊文件 (block special file)

&emsp;&emsp;这种类型的文件提供对设备 (如磁盘) 带缓冲的访问，每次访问以固定长度为单位进行。

4.   字符特殊文件 (character special file)

&emsp;&emsp;这种类型的文件提供对设备不带缓冲的访问，每次访问长度可变。系统中所有的设备要么是字符特殊文件，要么是块特殊文件。

>   FreeBSD 不再支持块特殊文件，对设备的所有访问都需要通过字符特殊文件进行。

5.   FIFO

&emsp;&emsp;这种类型的文件用于进程间通信，有时也被称为`命名管道` (named pipe)。

6.   套接字 (socket)

&emsp;&emsp;这种类型的文件用于进程间的网络通信。套接字也可用于在一台宿主机上进程之间的非网络通信。

7.   符号链接 (symbolic link)

&emsp;&emsp;这种类型的文件指向另一个文件。

-----

&emsp;&emsp;文件类型包含在 stat 结构中的 st_mode 成员中，可以用以下宏来检测，这些宏函数的参数都是 st_mode：

![](01.png)

&emsp;&emsp;POSIX 允许实现将进程间通信 (IPC) 对象 (如消息队列和信号量等) 说明为文件，下面的宏从 stat 结构中确定 IPC 对象的类型，但是它不以 st_mode 为参数，而是以 stat 结构指针为参数：

![](02.png)

&emsp;&emsp;下面的程序根据其命令行参数打印其文件类型：

```c
#include "apue.h"

int main(int argc, char **argv)
{
    char *ptr;
    struct stat buf;

    for (int i = 1; i < argc; i++) {
        printf("%s: ", argv[i]);
        
        if (lstat(argv[i], &buf) < 0) {
            err_ret("lstat error");
            continue;
        }

        if (S_ISREG(buf.st_mode))
            ptr = "regular";
        else if (S_ISDIR(buf.st_mode))
            ptr = "directory";
        else if (S_ISCHR(buf.st_mode))
            ptr = "character special";
        else if (S_ISBLK(buf.st_mode))
            ptr = "block special";
        else if (S_ISFIFO(buf.st_mode))
            ptr = "fifo";
        else if (S_ISLNK(buf.st_mode))
            ptr = "symbolic link";
        else if (S_ISSOCK(buf.st_mode))
            ptr = "socket";
        else
            ptr = "** unknown mode **";

        printf("%s\n", ptr);
    }
}
```

&emsp;&emsp;下面是输出示例：在 shell 中键入 \ 表示要在下一行继续键入命令。

```bash
$ ./a.out /etc/passwd /etc /dev/log /dev/tty \
> /var/lib/oprofile/opd_pipe /dev/sr0 /dev/cdrom
/etc/passwd: regular
/etc: directory
/dev/log: socket
/dev/tty: character special
/var/lib/oprofile/opd_pipe: fifo
/dev/sr0: block special
/dev/cdrom: symbolic link
```

## 设置用户 ID 和设置组 ID

&emsp;&emsp;与一个进程相关联的 ID 有 6 个甚至更多：

![](03.png)

-   实际用户 ID 和实际组 ID 标识进程的实际属主 (创建者)，这两个字段取自登录时口令文件中的登录项。通常，在一个登录会话期间这些值并不改变，但是超级用户有方法改变它们。
-   有效用户 ID 和有效组 ID 以及附属组 ID 决定了进程的文件访问权限。
-   保存的设置用户 ID 和保存的设置组 ID 在执行一个程序时包含了有效用户 ID 和有效组 ID 的副本。

&emsp;&emsp;通常，有效用户 ID 等于实际用户 ID，有效组 ID 等于实际组 ID。

&emsp;&emsp;每个文件有一个所有者和组所有者，所有者由结构 stat 中的 st_uid 指定，组所有者由 st_gid 指定。

&emsp;&emsp;当执行一个程序文件时，进程的有效用户和有效组 ID 通常就是实际用户和实际组 ID。但是文件可以在其`文件模式字` (st_mode) 中设置两个特殊标志：`设置用户 ID 位` (set-user-ID) 和 `设置组 ID 位` (set-group-ID)。当设置这两个标志的文件被执行时，执行进程的有效用户和有效组 ID 就被设置为此文件的所有者 (st_uid) 和所有组 (st_gid) ID。

&emsp;&emsp;例如，UNIX 系统程序 passwd(1) 允许任意用户改变其口令，但是只有超级用户才有权限对超级用户的口令文件 (etc/passwd) 写入。所以需要使用设置用户 ID 功能，使得运行该程序的进程获得额外的权限。

&emsp;&emsp;这两个特殊标志位都包含在文件的 st_mode 值中，分别用常量 S_ISUID 和 S_ISGID 测试。

## 文件访问权限

&emsp;&emsp;st_mode 值中也包含了对文件的访问权限，所有类型的文件都有 9 个访问权限位，可以分成三类：

![](04.png)

&emsp;&emsp;前三个权限中，用户指的是文件所有者 (owner)。chmod(1) 命令可以修改这 9 个权限位，该命令允许使用 u 表示用户 (所有者)，用 g 表示组，用 o 表示其他。

&emsp;&emsp;关于 9 个权限的使用规则：

-   当使用路径名打开任一类型的文件时，对该路径名中的每一个目录，包括它可能隐含的当前工作目录都应该具有执行权限。目录的读权限和执行权限不同，前者允许用户获得目录中的所有文件名的列表，而后者则是访问该目录下特定文件的必要权限。

&emsp;&emsp;例如：打开文件 /usr/include/stdio.h 需要对目录 /, /usr, /usr/include 都具有执行权限。并且还需要对文件 stdio.h 本身具有适当的权限。

-   对文件具有读权限允许进程打开文件进行读操作，与 open 函数中的标志 O_RDONLY 和 O_RDWR 相关。
-   对文件具有写权限允许进程打开文件进行写操作，与 open 函数中的标志 O_WRONLY 和 O_RDWR 相关。
-   使用 open 函数以 O_TRUNC 标志打开一个文件时，需要对文件有写权限。
-   在目录中创建一个新文件时，需要对该目录有写权限和执行权限。
-   删除一个现有文件时，需要对包含该文件的目录有写权限和执行权限。**但是对该文件本身不需要读写权限。**
-   如果使用 7 个 exec 函数执行某个文件，则必须对该文件具有执行权限，并且该文件还必须是一个普通文件。

-----

&emsp;&emsp;进程每次打开，创建或删除一个文件时，内核会进行文件访问权限测试。这种测试可能涉及文件所有者 (st_uid 和 st_gid) 以及进程的有效用户，有效组和附属组 ID。前者是文件的属性，后者是进程的属性。测试流程：

1.   若进程的有效用户 ID 为 0 (超级用户)，则允许访问。
2.   若进程的有效用户 ID 等于文件的所有者 ID (文件所有者和进程所有者相同)，那么进行下一步检测：若进程为读打开该文件，则用户读位应该为 1，若进程为写打开该文件，则用户写位应该为 1，若进程要执行该文件，则用户执行位应该为 1。以上规则满足则允许访问，否则拒绝访问。
3.   若进程的有效组 ID 或进程的某个附属组 ID 等于文件的组 ID，那么进行下一步检测 (以组权限进行匹配)。
4.   若其他用户适当的访问权限位被设置，则允许访问，否则拒绝访问。

&emsp;&emsp;以上四步按顺序匹配，如果进程拥有文件，则以文件的 3 个用户权限位检测。如果进程属于某个适当的组，则以文件的组权限位匹配。如果进程属于其他用户，则以文件的其他权限位匹配。

## 新文件和目录的所有权

&emsp;&emsp;当进程调用 open 或 creat 函数创建一个新文件时，新文件的用户 ID 被设置为进程的有效用户 ID。POSIX 允许新文件的组 ID 可以选择以下实现方案：

1.   新文件的组 ID 可以是进程的有效组 ID。
2.   新文件的组 ID 可以是它所在目录的组 ID。

>   FreeBSD 和 Max OS X 总是使用目录的组 ID 作为新文件的组 ID。对于 Linux 和 Solaris，默认情况下，新文件的组 ID 取决于它所在的目录的设置组 ID 是否被设置，如果该目录的该位被设置，则新文件的组 ID 就位目录的组 ID，否则新文件的组 ID 就位进程的有效组 ID。

&emsp;&emsp;继承目录的组 ID 使得在某个目录下创建的文件和目录都具有该目录的组 ID。于是文件和目录的组所有权从该点向下传递。

## 函数 access 和 faccessat

&emsp;&emsp;使用 open 函数打开一个文件时，内核以进程的有效用户和有效组 ID 位基础来测试其访问权限。如果进程想要以其实际用户和实际组 ID 来进行反问权限测试，可以使用 access 和 faccessat 函数：

```c
#include <unistd.h>

int access(const char *pathname, int mode);
int faccessat(int fd, const char *pathname, int mode, int flag);

/* Both return: 0 if OK, −1 on error */
```

&emsp;&emsp;如果测试文件已经存在，*mode* 就为 F_OK，否则 *mode* 就是下列常量的按位或：

![](05.png)

&emsp;&emsp;faccessat 函数的参数中，如果 *pathname* 为相对路径，且 *flag* 为 AT_FDCWD，那么就在当前工作目录下寻找测试文件。否则以 *fd* 指向的打开目录结合 *pathname* 寻找测试文件。

&emsp;&emsp;*flag* 参数可以改变 faccessat 的行为，如果 *flag* 设置为 AT_EACCESS，访问检查使用的是调用进程的有效用户和有效组 ID，而不是实际用户和实际组 ID。

&emsp;&emsp;下面是使用 access 的例子：

```c
#include "apue.h"
#include <fcntl.h>

int main(int argc, char **argv)
{
    if (argc != 2)
        err_quit("usage: a.out <pathname>");
    
    if (access(argv[1], R_OK) < 0)
        err_ret("access error for %s", argv[1]);
    else
        printf("read access OK\n");
    
    if (open(argv[1], O_RDONLY) < 0)
        err_ret("open error for %s", argv[1]);
    else
        printf("open for reading OK\n");
}
```

## 函数 uamsk

&emsp;&emsp;uamsk 函数为进程设置文件模式创建屏蔽字，并返回之前的值 (没有出错返回)。

```c
#include <sys/stat.h>

mode_t umask(mode_t cmask);

/* Returns: previous file mode creation mask */
```

&emsp;&emsp;其中 *cmask* 是之前 9 个文件访问权限位常量 (S_IRUSR, S_IWUSR 等) 的或组合。

&emsp;&emsp;在进程创建一个新文件或目录时，一定会使用文件模式创建屏蔽字。函数 open 和 creat 都有一个参数 *mode*，它指定新文件的访问权限位。而在文件模式创建屏蔽字中为 1 的位，在文件 *mode* 中的相应位一定被关闭。

&emsp;&emsp;下面的程序展示了 uamsk 函数和文件模式创建屏蔽字的用法：

```c
#include "apue.h"
#include <fcntl.h>

#define RWRWRW (S_IRUSR | S_IDUSR | S_IRGRP | S_IWGRP | S_IROTH | S_IWOTH)

int main(void)
{
    umask(0);
    if (creat("foo", RWRWRW) < 0)
        err_sys("creat error for foo");
    umask(S_IRGRP | S_IWGRP | S_IROTH | S_IWOTH);
    if (creat("bar", RWRWRW) < 0)
        err_sys("creat error for bar");
}
```

&emsp;&emsp;测试结果：

```bash
$ umask # first print the current file mode creation mask
002
$ ./a.out
$ ls -l foo bar
-rw------- 1 sar    0 Dec 7 21:20 bar
-rw-rw-rw- 1 sar    0 Dec 7 21:20 foo
$ umask # see if the file mode creation mask changed
002
```

&emsp;&emsp;UNIX 系统大多数用户不会更改他们的 umask 值，通常在登录时，shell 的启动文件会设置一次，然后再不改变。上面的测试中可以发现，子进程的 umask 值改变并不会影响父进程 (shell)。

&emsp;&emsp;下图是 umask 值的八进制数值，一个值代表要屏蔽一种权限：

![](06.png)

&emsp;&emsp;常见的几个 umask 值有：002 禁止其他用户写。022 禁止同组和其他用户写。 027 禁止同组写以及其他用户读写或执行。

## 函数 chmod, fchmod 和 fchmodat

&emsp;&emsp;这三个函数可用于改变现有文件的访问权限：

```c
#include <sys/stat.h>

int chmod(const char *pathname, mode_t mode);
int fchmod(int fd, mode_t mode);
int fchmodat(int fd, const char *pathname, mode_t mode, int flag);

/* All three return: 0 if OK, −1 on error */
```

&emsp;&emsp;chomod 函数在指定的文件上进行操作，而 fchmod 对已打开的文件进行操作。fchmodat 函数 chmod 行为类似，但是可以选择使用相对路径和用 *fd* 组合的方法指定文件，*flag* 的取 AT_SYMLINK_NOFOLLOW 或者 AT_SYMLINK_FOLLOW 可以决定是否跟随一个符号链接。

&emsp;&emsp;改变一个文件权限位的前提是进程的有效用户 ID 必须等于文件的所有者 ID，或者该进程拥有超级用户权限。下面是参数 *mode* 常量的按位或：

![](07.png)

&emsp;&emsp;除了之前 9 个文件访问权限位，还有 6 个：两个设置 ID 常量 (S_ISUID 和 S_ISGID)，保存正文常量 (S_ISVIX) 以及 3 个组合常量 (S_IRWXU, S_IRWXG 和 S_IRWXO)。

&emsp;&emsp;下面的程序修改之前文件 foo 和 bar 的权限位：

```c
#include "apue.h"

int main(void)
{
    struct stat buf;
    
    /* turn on set-group-ID and turn off group-execute */
    
    if (stat("foo", &buf) < 0)
        err_sys("stat error for foo");
    if (chomod("foo", (buf.st_mode & ~S_IXGRP | S_ISGID)) < 0)
        err_sys("chomod error for foo");
    
    /* set absolute mode to "rw-r---r---" */
    
    if (chmod("bar", S_IRUSR | S_IWUSR | S_IRGRP | S_IROTH) < 0)
        err_sys("chmod error for bar");
}
```

&emsp;&emsp;运行后，foo 和 bar 的权限位信息：

```bash
$ ls -l foo bar
-rw-r--r-- 1 sar    0 Dec 7 21:20 bar
-rw-rwSrw- 1 sar    0 Dec 7 21:20 foo
```

&emsp;&emsp;ls 显示 foo 文件权限位中的 S 表示其设置组 ID 位已经被打开，同时组执行已经被关闭。并且，ls 列出的时间和日期并未更新，这是因为 ls 只列出最后修改文件内容的时间，而 chmod 函数只更新 i 节点的最近一次更新时间。

## 黏着位

&emsp;&emsp;在早期的 UNIX 系统中，S_ISVTX 被称为`黏着位` (sticky bit)：如果一个可执行程序文件的这一位被设置，那么当程序第一次被执行，在其终止时，程序的正文 (机器指令) 的一个副本会被保存在交换区。这使得下次执行该程序时能较快的将其载入内存。对于通用的应用程序，如文件编辑器和 C 语言编译器，能提高程序载入速度。后来的 UNIX 系统称他为`保存正文位` (saved-text-bit)。较新的 UNIX 系统大多配置了虚拟存储系统以及快速文件系统，所以不需要再使用这种技术。

&emsp;&emsp;现今的系统扩展了黏着位的使用范围，Single UNIX Specification 允许针对目录设置黏着位。如果对目录设置了黏着位，只有对该目录拥有写权限并且满足下列条件之一，才能删除或重命名该目录下的文件：

-   拥有此文件。
-   拥有此目录。
-   是超级用户。

&emsp;&emsp;目录 /tmp 和 /var/tmp 是设置黏着位的代表 - 任何用户都可以在这两个目录中创建文件，任一用户对这两个目录的权限通常都是读，写和执行。但是用户不能删除或重命名其他人的文件。

## 函数 chown, fchown, fchownat 和 lchown

&emsp;&emsp;这几个 chown 函数用于更改文件的用户 ID 和组 ID，若参数 *owner* 或 *group* 为 -1 则对应的 ID 不变。

```c
#include <unistd.h>

int chown(const char *pathname, uid_t owner, gid_t group);
int fchown(int fd, uid_t owner, gid_t group);
int fchownat(int fd, const char *pathname, uid_t owner, gid_t group, int flag);
int lchown(const char *pathname, uid_t owner, gid_t group);

/* All four return: 0 if OK, −1 on error */
```

&emsp;&emsp;和之前的函数类似，除了 lchown 外，其他的函数都不跟随符号链接文件，即会修改符号链接指向的文件而不是符号链接本身。fchownat 函数可以在另外一个打开目录中寻找文件，和其他 at 类函数相同。

&emsp;&emsp;但是 chown 函数在很多 UNIX 系统上都有限制，即只允许 root 用户才能更改文件或目录的属主。

## 文件长度

&emsp;&emsp;stat 结构成员 st_size 表示以字节为单位的文件长度，此字段只对普通文件，目录文件和符号链接有意义。

&emsp;&emsp;对于普通文件，其文件长度可以是 0，在开始读这种文件时，将得到文件结束 (end-of-file) 指示。对于目录，文件长度通常是一个数 (如 16 或 512) 的整数倍。

&emsp;&emsp;对于符号连接，文件长度是文件名的实际字节数，文件长度 7 就是路径名的 usr/lib 的长度：

```bash
$ ls -l /lib
lrwxrwxrwx. 1 root root 7 Jan 19  2023 /lib -> usr/lib
```

&emsp;&emsp;大多数现代的 UNIX 系统提供字段 st_blksize 和 st_blocks。其中，第一个是对文件 I/O 较合适的块长度，第二个是所分配的实际 512 字节块块数。通常，将 st_blksize 用于读操作时，读一个文件所需的时间最少。为了提高效率，标准 I/O 也试图一次读，写 st_blksize 个字节。

-----

&emsp;&emsp;文件中的空洞是由于所设置的偏移量超过了文件尾端，并写入数据造成的，一个例子：

```bash
$ ls -l core
-rw-r--r-- 1 sar    8483248 Nov 18 12:18 core
$ du -s core
272     core
```

&emsp;&emsp;文件 core 的长度稍超过 8 MB，可是 du 命令显示文件所用的磁盘空间总量是 272 个 512 字节块。

&emsp;&emsp;当读取一个文件的空洞时，read 函数读到的字节都是 0。如果使用实用程序 cat(1) 复制这个文件，那么所有的这些空洞都会被填满，其中所有实际数据字节皆填写为 0。

```bash
$ cat core > core.copy
$ ls -l core*
-rw-r--r-- 1 sar 8483248 Nov 18 12:18 core
-rw-rw-r-- 1 sar 8483248 Nov 18 12:27 core.copy
$ du -s core*
272   core
16592 core.copy
```

&emsp;&emsp;此时新文件 core.copy 已经没有空洞了，但是其实际所用的字节数是 8 495 104，与 ls 显示的并不相同。这是因为，文件系统使用了若干块以存放指向实际数据块的各个指针。

## 文件截断

&emsp;&emsp;将文件截断到 0 可以使用 open 函数中的 O_TRUNC 标志。并且使用 truncate 和 ftruncate 函数可以从文件的任一位置开始截断：

```c
#include <unistd.h>

int truncate(const char *pathname, off_t length);
int ftruncate(int fd, off_t length);

/* Both return: 0 if OK, −1 on error */
```

&emsp;&emsp;这两个函数将一个现有文件截断为 *length*。如果该文件以前的长度大于 *length*，则超过 *length* 以外的数据就不能再访问。如果以前的长度小于 *length*，文件长度将增加，但是文件以前的末尾到 *length* 中间将创建一个空洞。

## 文件系统

&emsp;&emsp;目前，正在使用的 UNIX 文件系统有多种实现：传统的基于 BSD 的 UNIX 文件系统 (UFS)，读写 DOS 格式软盘的文件系统 (PCFS)，以及读 CD 的文件系统 (HSFS)。

&emsp;&emsp;下面的分析基于 UFS 文件系统。可以把一个磁盘分成一个或多个分区，每个分区可以包含一个文件系统。i-node 是固定长度的记录项，它包含有关文件的大部分信息：

![](08.png)

&emsp;&emsp;柱面组的 i-node 和数据块部分详细展示如下：

![](09.png)

-   图中有两个目录项指向同一个 i-node，每个 i-node 都有一个链接计数，其值是指向该 i-node 的目录项数。只有当链接计数减少到 0 时，才可以删除一个文件 (即释放该文件占用的数据块)。在 stat 结构中，链接计数包含在 st_nlink 成员中，其基本系统数据类型是 nlink_t。这种链接称为硬链接，POSIX 常量 LINK_MAX 指定了一个文件连接数的最大值。
-   另一种链接类型称为`符号链接` (symbolic link)。符号链接文件的实际内容包含了该符号链接所指向的文件的名字。之前使用 ls 列出的 /lib 目录就是一个符号链接。该 i-node 中的文件类型是 S_IFLNK。
-   i-node 包含了文件有关的所有信息：文件类型，文件访问权限位，文件长度和指向文件数据块的指针等。stat 结构中的大多数信息都取自 i-node。只有两项重要的数据存放在目录项中：文件名和 i-node 编号。i-node 编号的数据类型是 ino_t。
-   因为目录项中的 i-node 编号指向同一文件系统中的相应 i-node，一个目录项不能指向另一个文件系统的 i-node。这也是 ln(1) 命令不能跨越文件系统的原因。
-   当在不更换文件系统的情况下为一个文件重命名时，该文件的实际内容并未移动，只需构造一个指向现有 i-node 的新目录项，并删除旧目录项。链接计数并不会改变，这也是 mv(1) 命令的通常操作方式。

-----

&emsp;&emsp;目录文件的链接计数字段略有不同，先在工作目录中创建一个子目录：

```bash
$ mkdir testdir
```

![](10.png)

&emsp;&emsp;编号为 2549 的 i-node，其类型字段显示它是一个目录，链接计数为 2。任何一个空的叶目录的链接计数总是 2：一个链接来自其父目录项，另一个链接来自其自己目录项中的 . 条目。编号为 1267 的 i-node 也是一个目录，并且其链接计数为 3：一个链接来自其父目录项，一个链接来自其自己目录项中 . 条目，另一个链接来自 testdir 目录项中的 .. 条目。

## 函数 link, linkat, unlink, unlinkat 和 remove

&emsp;&emsp;创建一个指向现有文件的链接的方法是使用 link 函数或 linkat 函数：

```c
#include <unistd.h>

int link(const char *existingpath, const char *newpath);
int linkat(int efd, const char *existingpath, int nfd, const char *newpath, int flag);

/* Both return: 0 if OK, −1 on error */
```

&emsp;&emsp;这两个函数创建一个新的目录项 *newpath*，它引用现有文件 *existingpath*。如果 *newpath* 已经存在，则返回出错。只创建 *newpath* 中的最后一个分量，路径中的其他部分应当已经存在。

&emsp;&emsp;对于 linkat 函数，现有文件是通过 *efd* 和 *existingpath* 指定的。新路径名是通过 *nfd* 和 *newpath* 指定的。路径名的计算类似于其他 at 类函数。当现有文件是符号连接时，由 *flag* 参数来控制 linkat 函数是创建指向现有符号链接的链接还是创建指向该符号连接目标的链接。

&emsp;&emsp;创建新目录项和增加链接计数应当是一个原子操作。虽然 POSIX 允许实现支持跨越文件系统的链接，但是大多数实现要求现有的和新建的两个路径名在同一个文件系统中，原因这样做可能在文件系统中形成循环。因此，很多文件系统实现不允许对目录的硬链接。

&emsp;&emsp;删除一个已有的目录项，可以调用 unlink 函数：

```c
#include <unistd.h>

int unlink(const char *pathname);
int unlinkat(int fd, const char *pathname, int flag);

/* Both return: 0 if OK, −1 on error */
```

&emsp;&emsp;这两个函数删除目录项，并将由 *pathname* 所引用文件的链接计数减 1，如果对该文件还有其他链接，则仍可以通过其他链接访问该文件的数据。如果出错，则不对该文件做任何事。

&emsp;&emsp;删除一个目录项，要求对于包含该目录项的目录有执行和写权限。如果该目录设置了黏着位，除了对该目录有写权限外，还必须拥有该文件或者该目录，或者是超级用户。

&emsp;&emsp;只有当文件链接计数达到 0 时，该文件的内容才能被删除。但是，如果有进程打开了该文件，其内容也不能删除。关闭一个文件时，内核首先检查打开该文件的进程的个数，如果这个计数达到 0，内核再去检查其链接计数：如果链接计数也是 0，那么就删除该文件的内容。

&emsp;&emsp;unlinkat 函数计算文件路径名的方式类似其他 at 类函数。其 *flag* 参数可使调用进程改变 unlinkat 函数的默认行为。当 AT_REMOVEDIR 标志被设置时，unlinkat 函数类似于 rmdir 一样删除目录，否则，unlinkat 与 unlink 执行同样的操作。

-----

&emsp;&emsp;下面的程序打开一个文件，然后解除它的链接，执行该程序然后睡眠 15 秒：

```c
#include "apue.h"
#include <fcntl.h>

int main(void)
{
    if (open("tempfile", O_RDWR) < 0)
        err_sys("open error");
    if (unlink("tempfile") < 0)
        err_sys("unlink error");
    printf("file unlinked\n");
    sleep(15);
    printf("done\n");
}
```

&emsp;&emsp;运行该程序：

![](11.png)

&emsp;&emsp;如果 *pathname* 是符号连接，那么 unlink 删除该符号连接，而不是由其引用的文件。给出符号链接名的情况下，没有一个函数能删除其引用的文件。

&emsp;&emsp;如果文件系统支持的话，超级用户可以调用 unlink 删除一个目录。但是通常应该使用 rmdir 函数。

&emsp;&emsp;使用 remove 函数解除对一个文件或目录的链接，对于文件，其行为与 unlink 相同，对于目录，其行为与 rmdir 相同：

```c
#include <stdio.h>

int remove(const char *pathname);

/* Returns: 0 if OK, −1 on error */
```

## 函数 rename 和 renameat

&emsp;&emsp;目录或文件可以用 rename 和 renameat 函数进行重命名：

```c
#include <stdio.h>

int rename(const char *oldname, const char *newname);
int renameat(int oldfd, const char *oldname, int newfd, const char *newname);

/* Both return: 0 if OK, −1 on error */
```

&emsp;&emsp;根据 *oldname* 的类型，以及 *newname* 存在，有以下情况：

1.   如果 *oldname* 是一个文件而不是目录，那么为该文件或符号链接重命名。如果 *newname* 存在，则其不能引用一个目录。此时会先将 *newname* 目录项删除，然后将 *oldname* 重命名为 *newname*。进程必须对包含 *oldname* 和 *newname* 的两个目录都有写权限。
2.   如果 *oldname* 是一个目录，那么为该目录重命名。如果 *newname* 已经存在，则它必须是一个空目录 (只包含 . 和 ..)。此时会先将 *newname* 删除，然后将 *oldname* 重命名为 *newname*。并且，*newname* 不能包含 *oldname* 作为其路径前缀。
3.   如果 *oldname* 或 *newname* 引用符号链接，则处理符号链接本身，而不是其引用的文件。
4.   不能对 . 和 .. 重命名，即 . 和 .. 不能出现在 name 的最后部分。
5.   如果 *oldname* 和 *newname* 引用同一文件，则函数不做任何更改并返回成功。

&emsp;&emsp;如果 *newname* 已经存在，则调用进程对它要有写权限。并且因为涉及两个文件，所以调用进程必须对包含 *oldname* 和 *newname* 的目录都有写权限和执行权限。

## 符号链接

&emsp;&emsp;符号链接是对一个文件的间接指针，它与硬链接不同，硬链接是直接指向文件的 i-node。引入符号链接的是为了避开硬链接的一些限制：

-   硬链接通常要求链接和文件位于同一文件系统。
-   只有超级用户才能创建指向目录的硬链接 (文件系统支持的话)。

&emsp;&emsp;对符号链接以及它指向的对象类型无任何文件系统限制，任何用户都可以创建指向目录的符号链接。符号链接一般用于将一个文件或整个目录结构移到系统中的另一个位置。

&emsp;&emsp;使用以路径名为参数的函数时，要考虑其是否跟随符号链接：

![](12.png)

&emsp;&emsp;使用符号链接可能在文件系统中引入循环，大多数查找路径名的函数在这种情况发生时都将出错返回，errno 值为 ELOOP：

```bash
$ mkdir foo                  # make a new directory
$ touch foo/a                # create a 0-length file
$ ln -s ../foo foo/testdir   # create a symbolic link
$ ls -l foo
total 0
-rw-r----- 1 sar 0 Jan 22 00:16 a
lrwxrwxrwx 1 sar 6 Jan 22 00:16 testdir -> ../foo
```

&emsp;&emsp;这创建了一个目录 foo，它包含一个名为 a 的文件以及一个指向 foo 的符号链接：

![](13.png)

&emsp;&emsp;如果使用 ftw(3) 标准函数遍历文件结构，其输出会是：

```bash
foo
foo/a
foo/testdir
foo/testdir/a
foo/testdir/testdir
foo/testdir/testdir/a
foo/testdir/testdir/testdir
foo/testdir/testdir/testdir/a
(many more lines until we encounter an ELOOP error)
```

&emsp;&emsp;这样的循环很容易消除，因为 unlink 并不跟随符号链接，所以直接删除文件 foo/testdir。但是如果创建了一个这样循环的硬链接，那么就很难消除它。

&emsp;&emsp;用 open 打开文件时，如果传递的路径名是一个符号链接，那么 open 跟随此链接到达所指定的文件，若符号链接所指向的文件不存在，则 open 返回出错。下面是一个常见错误：

![](14.png)

&emsp;&emsp;这是因为 myfile 存在，但是 myfile 指向的文件却不存在。

## 创建和读取符号链接

&emsp;&emsp;可以使用 symlink 或 symlinkat 函数创建一个符号链接：

```c
#include <unistd.h>

int symlink(const char *actualpath, const char *sympath);
int symlinkat(const char *actualpath, int fd, const char *sympath);

/* Both return: 0 if OK, −1 on error */
```

&emsp;&emsp;函数创建一个指向 *actualpaht* 的新目录项 *sympath*。在创建符号链接时，并不要求 *actualpath* 已经存在。并且，*actualpath* 和 *sympath* 不需要位于同一文件系统中。

&emsp;&emsp;因为 open 函数跟随符号链接，所以需要有一种方法打开链接本身，并读取该链接中的名字：

```c
#include <unistd.h>

ssize_t readlink(const char* restrict pathname, char *restrict buf, size_t bufsize);
ssize_t readlinkat(int fd, const char* restrict pathname, char *restrict buf, size_t bufsize);

/* Both return: number of bytes read if OK, −1 on error */
```

&emsp;&emsp;这两个函数组合了 open, read 和 close 的所有操作。如果函数成功执行，则返回读入 *buf* 的字节数，在 *buf* 中返回的符号链接的内容不以 NULL 字符终止。

## 文件的时间

&emsp;&emsp;文件属性中保存的实际时间精度依赖于文件系统的实现。对于把时间戳记录在秒级的文件系统来说，纳秒这个字段会被填充为 0。对于时间戳的记录精度高于秒级的系统来说，不足秒的 值会被转换成纳秒并记录在纳秒字段中。

&emsp;&emsp;文件系统为每个文件维护 3 个时间字段：

![](15.png)

&emsp;&emsp;修改时间 st_mtim 和状态更改时间 st_ctim 是不同的，修改时间是文件内容最后一次被修改的时间。而状态更改时间，是该文件 i-node 最后一次被修改的时间。很多操作，如更改文件访问权限，更改用户 ID，更改链接数等都会修改 i-node 内容，但是他们并不更改文件内容，这两部分数据是分开放的。

&emsp;&emsp;影响这 3 个时间的函数表：

![](16.png)

## 函数 futimens, utimensat 和 utimes

&emsp;&emsp;一个文件的访问和修改时间可以用以下几个函数修改。futimens 和 utimensat 函数可以指定纳秒级别精度的时间戳，用到的数据结构是 timespec 结构：

```c
#include <sys/stat.h>

int futimens(int fd, const struct timespec times[2]);
int utimensat(int fd, const char *path, const struct timespec times[2], int flag);

/* Both return: 0 if OK, −1 on error */
```

&emsp;&emsp;这两个函数的 *times* 数据的第一元素包含访问时间，第二个元素包含修改时间。这两个时间值是日历时间，即自世界协调时依赖所经过的秒数，不足秒的部分用纳秒表示。

&emsp;&emsp;时间戳可以按以下 4 种方式之一指定：

1.   如果 *times* 参数是一个空指针，则访问时间和修改时间都设置为当前时间。
2.   如果 *times* 参数指向两个 timespec 结构数组，任一数组元素的 tv_nsec 字段的值为 UTIME_NOW，相应的时间戳就设置为当前时间，忽略相应的 tv_sec 字段。
3.   如果 *times* 参数指向两个 timespec 结构数组，任一数组元素的 tv_nsec 字段的值为 UTIME_OMIT，相应的时间戳保持不变，忽略相应的 tv_sec 字段。
4.   如果 *times* 参数指向两个 timespec 结构数组，且 tv_nsec 字段的值不是以上两个常量，那么相应的时间戳就设置为相应的 tv_sec 和 tv_nsec 的值。

&emsp;&emsp;执行这些函数的优先权取决于 *times* 参数的值：

-   如果 *times* 是空指针，或者任一 tv_nsec 字段为 UTIME_NOW，则进程的有效用户 ID 必须等于该文件的所有者 ID：进程必须对该文件有写权限，或者是有超级用户权限。
-   如果 *times* 不是空指针，并且 tv_nsec 的值不是上面两个常量，则进程的有效用户 ID 必须等于文件所有者的 ID，对文件只有写权限是不够的。
-   如果 *times* 不是空指针，而且两个元素的 tv_nsec 字段都是 UTIME_OMIT，就不执行任何权限检查。

&emsp;&emsp;futimens 函数需要打开文件来更改它的时间，utimensat 函数可以使用文件名，并且 *path* 是相对打开目录 *fd* 来计算的。计算细节类似其他 at 类函数。并且 *flag* 参数可以控制是否跟随符号链接文件。

&emsp;&emsp;上面两个函数是 POSIX 所包含的，第 3 个函数 utimes 包含在 XSI 扩展中：

```c
#include <sys/time.h>

int utimes(const char *pathname, const struct timeval times[2]);

/* Returns: 0 if OK, −1 on error */
```

&emsp;&emsp;其中 timeval 结构类似 timespec 结构，也包含秒和纳秒两个字段。

&emsp;&emsp;注意，不能更改状态时间 st_ctim (i-node 最近修改时间)，这是因为在修改其他两个时间的时候，此字段会被自动更新。

-----

&emsp;&emsp;下面的程序先使用 stat 函数得到文件的修改时间和访问时间，然后使用 O_TRUNC 标志截断文件，最后使用 futimens 函数重置这两个时间为之前的时间：

```c
#include "apue.h"
#include <fcntl.h>

int main(int argc, char *argv[])
{
    int i, fd;
    struct stat statbuf;
    struct timespec times[2];
    for (i = 1; i < argc; i++) {
        if (stat(argv[i], &statbuf) < 0) { /* fetch current times */
            err_ret("%s: stat error", argv[i]);
            continue;
        }
        if ((fd = open(argv[i], O_RDWR | O_TRUNC)) < 0) { /* truncate */
            err_ret("%s: open error", argv[i]);
            continue;
        }
        times[0] = statbuf.st_atim;
        times[1] = statbuf.st_mtim;
        if (futimens(fd, times) < 0)
            /* reset times */
            err_ret("%s: futimens error", argv[i]);
        close(fd);
    }
}
```

&emsp;&emsp;测试结果如下：

![](17.png)

&emsp;&emsp;最后，文件的修改时间和访问时间都没改变。但是，状态更改时间更改为程序运行的时间。

## 函数 mkdir, mkdirat 和 rmdir

&emsp;&emsp;用 mkdir 和 mkdirat 函数创建目录，用 rmdir 删除目录：

```c
#include <sys/stat.h>

int mkdir(const char *pathname, mode_t mode);
int mkdirat(int fd, const char *pathname, mode_t mode);

/* Both return: 0 if OK, −1 on error */
```

&emsp;&emsp;这两个函数创建一个新的空目录，其中 . 和 .. 是自动创建的。所指定的文件访问权限是由 *mode* 和进程的文件模式创建屏蔽字修改。

&emsp;&emsp;常见的错误是只指定读写权限，但是，对于目录至少要设置一个执行权限位，以允许访问目录中的文件名。

&emsp;&emsp;如果父目录设置了 set-user-ID 位或 set-group-ID 位，那么新目录将继承父目录的用户 ID 或组 ID。否则，新目录将使用进程的有效用户 ID 和有效组 ID。这对新文件也适用。

&emsp;&emsp;用 rmdir 函数可以删除一个空目录：

```c
#include <unistd.h>

int rmdir(const char *pathname);

/* Returns: 0 if OK, −1 on error */
```

&emsp;&emsp;如果调用此函数使目录的链接计数成为 0，并且没有其他进程打开此目录，则释放由此目录占用的磁盘空间。如果在链接计数达到 0 时，有一个或多个进程打开此目录，则在此函数返回前删除最后一个链接及 . 和 .. 项。另外，在此目录中不能再新建文件，但是在最后一个进程关闭它之前并不释放该目录。(即使另一些进程打开该目录，它们也不能在此目录执行其他操作。这是因为 rmdir 要求删除的目录必须是一个空目录。)

## 读目录

&emsp;&emsp;对某个目录具有访问权限的任一用户都可以读该目录，但是，为了防止文件系统产生混乱，只有内核才可以写目录。一个目录的写权限位和执行权限位决定了在该目录中能否创建和删除文件，但并不代表能否写目录本身。

&emsp;&emsp;目录的实际格式依赖于 UNIX 系统实现和文件系统的设计。早期的系统有一个简单的结构：每个目录项是 16 个字节，其中 14 字节存放文件名，2 个字节存放 i-node 编号。而新的系统则允许更长的文件名，所以每个目录项的长度是可变的。这就意味着读目录的程序与系统相关，为了简化读目录的过程，UNIX 包含了一套与目录有关的例程，它们是 POSIX 的一部分。很多实现阻止程序使用 read 函数读取目录的内容，来进一步隔离程序与目录格式的实现：

```c
#include <dirent.h>

DIR *opendir(const char *pathname);
DIR *fdopendir(int fd);
/* Both return: pointer if OK, NULL on error */

struct dirent *readdir(DIR *dp);
/* Returns: pointer if OK, NULL at end of directory or error */

void rewinddir(DIR *dp);
int closedir(DIR *dp);
/* Returns: 0 if OK, −1 on error */

long telldir(DIR *dp);
/* Returns: current location in directory associated with dp */

void seekdir(DIR *dp, long loc);
```

&emsp;&emsp;fdopendir 函数提供将打开的文件描述符转换成 DIR 结构的功能。

&emsp;&emsp;telldir 和 seekdir 函数不是 POSIX 标准的组成部分，它们属于 XSI 扩展。

&emsp;&emsp;dirent 结构与实现有关，但是它至少包含两个成员：

```c
ino_t d_ino;       /* i-node number */
char d_name[256];  /* null-terminated filename */
```

&emsp;&emsp;d_name 的大小可以不指向，256 是 Linux 上的定义，并且文件名是以 NULL 字符结尾的。

&emsp;&emsp;DIR 结构是一个内部结构，上面 7 个函数使用这个内部结构保存当前正在被读的目录的有关信息。

&emsp;&emsp;opendir 和 fdopendir 函数执行初始化操作，readdir 返回目录中的当前目录项。目录中各目录项的顺序与实现有关，通常并不是按字母顺序排列的。

## 函数 chdir, fchdir 和 getcwd

&emsp;&emsp;每个进程都有一个当前工作目录，此目录是搜索所有相对路径的起点。当用户登录到 UNIX 系统时，其当前工作目录通常是口令文件中该登录项的第 6 个字段。

&emsp;&emsp;进程可以调用 chdir 或 fchdir 函数更改当前工作目录：

```c
#include <unistd.h>

int chdir(const char *pathname);
int fchdir(int fd);

/* Both return: 0 if OK, −1 on error */
```

&emsp;&emsp;当前工作目录是进程的一个属性，它只影响调用 chdir 的进程本身，而不影响其他进程：

```c
#include "apue.h"

int main(void)
{
    if (chdir("/tmp") < 0)
        err_sys("chdir failed");
    printf("chdir to /tmp succeeded\n");
}
```

&emsp;&emsp;在 shell 中测试：

```bash
$ pwd
/usr/lib
$ mycd
chdir to /tmp succeeded
$ pwd
/usr/lib
```

&emsp;&emsp;执行 mycd 命令的 shell 的当前工作目录并未改变，这是因为：每个程序运行在独立的进程中，shell 的当前工作目录不会随着程序改变而改变。要改变 shell 的工作目录，shell 应直接调用 chdir 函数。

&emsp;&emsp;因为内核必须维护当前工作目录的信息，所以进程应当能获取其值。但是，内核为每个进程值保存其 v-node 信息，而非目录项信息，所以路径名并不被直接保存。但是通过特殊的办法，可以使用 getcwd 函数获取：

```c
#include <unistd.h>

char *getcwd(char *buf, size_t size);

/* Returns: buf if OK, NULL on error */
```

&emsp;&emsp;其中，*buf* 是缓冲区地址，*size* 是缓冲区长度。该缓冲区必须有足够的长度以容纳绝对路径名加上一个 NULL 字符，否则返回出错。

## 块设备特殊文件

&emsp;&emsp;编写 ttyname 函数时，需要使用 stat 结构中的 st_dev 和 st_rdev 字段，有关规则：

-   每个文件系统所在的存储设备都由其主，次设备号表示。设备号所用的数据类型是 dev_t：主设备号标识设备驱动程序，有时编码为与其通信的外设版，次设备号标识特定的子设备。一个磁盘驱动器经常包含若干个文件系统，在同一磁盘驱动器上个各文件系统有相同的主设备号，但是次设备号却不同。
-   可以使用两个宏：major 和 minor 来访问主，次设备号，而无需关心这两个数是如何存储在 dev_t 对象中的。
-   系统中与每个文件关联的 st_dev 值是文件系统的设备号，该文件系统包含了这一文件名以及与其对应的 i-node。
-   只有字符特殊文件和块特殊文件才有 st_rdev 值，次值包含实际设备的设备号。

&emsp;&emsp;下面的程序会打印命令行参数的设备号，并且还会区分字符或块特殊文件：

```c
#include "apue.h"
#ifdef SOLARIS
#include <sys/mkdev.h>
#endif

int main(int argc, char *argv[])
{
    int i;
    struct stat buf;
    for (i = 1; i < argc; i++) {
        printf("%s: ", argv[i]);
        if (stat(argv[i], &buf) < 0) {
            err_ret("stat error");
            continue;
        }
        printf("dev = %d/%d", major(buf.st_dev), minor(buf.st_dev));
        if (S_ISCHR(buf.st_mode) || S_ISBLK(buf.st_mode)) {
            printf(" (%s) rdev = %d/%d",
                   (S_ISCHR(buf.st_mode)) ? "character" : "block",
                   major(buf.st_rdev), minor(buf.st_rdev));
        }
        printf("\n");
    }
}
```

&emsp;&emsp;测试结果：

![](18.png)

&emsp;&emsp;其中最后一个命令行参数使用 shell 正则表达式，shell 会将字符串 /dev/tty[01] 扩展为 /dev/tty0 和 /dev/tty1。

&emsp;&emsp;结果显示，根目录 / 和 /home/sar 目录的次设备号不同，这表示它们位于不同的文件系统中，mount(1) 命令证明这一点。使用 ls 命令查看两个磁盘设备和两个终端设备。通常，只有包含随机访问文件系统的设备是块特殊文件设备，如硬盘驱动器，软盘驱动器和 CD-ROM 等。

&emsp;&emsp;两个终端设备 st_dev 的文件名和 i-node 在设备 0/5 上 (devtmpfs 伪文件系统实现了 /dev 文件系统)，但是它们的实际设备号 st_rdev 是 4/0 和 4/1。

## 文件反问权限位小节

&emsp;&emsp;下图列出文件权限位常量以及分别对文件和目录的作用：

![](19.png)

&emsp;&emsp;最后 9 个常量还能分成 3 组：

```c
S_IRWXU = S_IRUSR | S_IWUSR | S_IXUSR
S_IRWXG = S_IRGRP | S_IWGRP | S_IXGRP
S_IRWXO = S_IROTH | S_IWOTH | S_IXOTH
```