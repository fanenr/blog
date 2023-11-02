---
title: CSAPP 10 - 系统级 I/O
date: 2023-08-21 18:35:31
updated: 2023-08-24 19:00:00
tags:
    - CSAPP
    - C
categories:
    - CSAPP
---

&emsp;&emsp;I/O 就是在主存和外部设备之间复制数据的过程。

<!-- more -->

## UNIX I/O

&emsp;&emsp;一个 Linux 文件就是一个 m 个字节的序列：B~0~ B~1~ ... B~m-1~。所有的 I/O 设备都被模型话为文件，而所有的输入输出都被当作对相应文件的读和写来执行。这种将设备映射为文件的方式，允许 Linux 内核引出一个简单，低级的应用接口，称为 Unix I/O。它使得所有的输入输出都能以一种统一且一致的方式来执行：

*   打开文件

&emsp;&emsp;应用程序通过向内核发起打开文件请求以访问 I/O 设备。内核将返回一个小的非负整数，叫做`文件描述符`，它将在对文件的后续操作中标识该文件。内核跟踪与打开文件相关的所有信息，而应用程序则只跟踪描述符。每个 shell 创建的进程都会打开三个文件：标准输入`STDIN_ FILENO`，描述符为 0，标准输出`STDOUT_FILENO`，描述为 1，标准错误`STDERR_FILENO`，描述符为 2。

*   改变当前的文件位置

&emsp;&emsp;内核为每个打开文件维护一个文件位置 k，初始值为 0。文件位置是文件中下一个即将被读取或写入的字符到文件起始位置的字节偏移量，并非指该文件在文件系统中的位置。应用程序可以通过执行 seek 操作来显式地设置当前文件位置 k。

*   读写文件

&emsp;&emsp;读取操作从当前文件位置 k 开始，复制 n 个字节到内存中，随后令 k 增加 n。当文件位置大于或等于文件大小时，读取操作会触发 EOF。类似地，写入操作从当前文件位置 k 开始，复制 n 个字节到文件中并更新 k 的值。

*   关闭文件

&emsp;&emsp;当应用程序结束对文件的访问时，它会向内核发起关闭文件请求，内核释放打开文件时创建的数据结构并将描述符放回可用描述符池。若进程因某些原因终止，内核将关闭所有打开的文件并释放相应的内存资源。

## 文件

&emsp;&emsp;每个 Linux 文件都有一个类型：

*   普通文件

&emsp;&emsp;对于应用程序来说，常规文件分为仅包含 ASCII 或 Unicode 字符的文本文件和二进制文件。但对于内核而言，两者没有区别。Linux 文本文件由一系列文本行组成，其中每一行都以换行符`\n`结尾。

*   目录

&emsp;&emsp;目录是包含一组链接的文件。其中每个链接将一个文件名映射到一个文件，该文件可能是另一个目录。每个目录中至少包含两个链接：`.`指向目录本身，而`..`指向上级目录。

*   套接字

&emsp;&emsp;套接字是用来与另一个进程进行跨网络通信的文件。

&emsp;&emsp;其他类型的文件包含`命名管道`，`符号链接`，以及`字符和块设备`。

&emsp;&emsp;Linux 内核将所有的文件都组织成一个`目录层次结构`，由根目录`/`确定。系统中每个文件都是根目录的直接或间接后代：

![](01.png)

&emsp;&emsp;每个进程都有一个`当前工作目录`来确定其在目录层次结构中的当前位置，在 shell 中，可以使用 cd 命令来修改当前工作目录。

## 打开和关闭文件

&emsp;&emsp;进程通过调用 open 函数来打开一个已存在的文件或创建一个新文件：

```c
int open(char *filename, int flags, mode_t mode);
```

&emsp;&emsp;open 函数将 filename 转换为一个文件描述符，并返回其数字。返回的描述符总是在进程中当前没有打开的最小描述符。flags 参数指明了进程打算如何访问这个文件：

*   O_RDONLY：只读
*   O_WRONLY：只写
*   O_RDWR：可读可写

&emsp;&emsp;flags 也可以是一个或者更多位掩码的或，为写提供一些额外的指示：

*   O_CREAT：如果文件不存在，则创建一个`截断它`的空文件。
*   O_TRUNC：如果文件已存在，就截断它。
*   O_APPEND：在每次写操作前，设置文件位置到文件的结尾处。

&emsp;&emsp;mode 参数指定了新文件的访问权限位：

![](02.png)

&emsp;&emsp;作为上下文的一部分，每个进程都有一个 umask，它是通过调用 umask 函数来设置的。当进程通过某个带 mode 参数的 open 函数来创建新文件时，文件的访问权限会被设置为 mode & ~umask。

&emsp;&emsp;最后，进程通过调用 close 函数来关闭一个打开的文件：

```c
int close(int fd);
```

&emsp;&emsp;注意：关闭一个已关闭的文件描述符会出错。

## 读写文件

&emsp;&emsp;程序通过 read 和 write 函数来执行输入和输出：

```c
ssize_t read(int fd, void *buf, size_t n);
ssize_t write(int fd, const void *buf, size_t n);
```

&emsp;&emsp;read 函数从描述符为 fd 的文件的当前位置复制最多 n 个字节到内存位置 buf。返回 -1 表示遇到一个错误，返回值 0 代表 EOF。返回正数代表实际传送的字节数量。

&emsp;&emsp;write 函数从内存位置 buf 复制最多 n 个字节到文件描述为 fd 的文件的当前位置。

&emsp;&emsp;在 x86-64 系统中，size_t 被定义为 unsigned long，而 size_t 被定义为 long。这是因为 read 和 write 函数都可能返回负数 -1 来表示错误。但这也使得 read 和 write 的最大值减小了一半。

&emsp;&emsp;在某些情况下，读写操作传输的字节数会小于应用程序请求的字节数。这些`不足值`的产生并不代表发生了错误，它可能由多种原因导致：

-   读取时遇到 EOF：若对一个 20 字节的文件执行 read(..., 50)，那么第一次调用将返回一个 20 的不足值，第二次调用则返回 0。
-   从终端读取文本行：若打开的文件是终端设备 (键盘和显示器)，那么每次 read 调用都将传输一个文本行并返回一个与文本行大小相等的不足值。
-   读写套接字：若打开的文件是套接字，那么内部缓冲区限制和网络延迟将使读写操作返回不足数。

&emsp;&emsp;实际上，除遇到 EOF 外，读写磁盘文件不会导致不足数的产生。但如果想要构建健壮而可靠的网络应用程序，就必须重复调用 read 和 write 以保证所有请求的字节均已被传输。

## TODO RIO 包

&emsp;&emsp;待更新。

## 读取文件元数据

&emsp;&emsp;程序可以通过调用 stat 和 fstat 函数，检索到关于文件的信息，也就是元数据：

```c
int stat(const char *filename, struct stat *buf);
int fstat(int fd, struct stat *buf);
```

&emsp;&emsp;函数 stat 使用文件名 filename 作为输入，将信息填写到类型为 stat 的 buf 结构体中。fstat 与之类似，但它的参数是文件描述符fd。stat 结构体的成员：

![](03.png)

&emsp;&emsp;st_size 成员包含文件的字节大小，st_mode 成员编码了文件的访问许可位。Linux 提供了宏谓词来确定 st_mode 成员的文件类型：

*   S_ISREG(m)：是否是普通文件
*   S_ISDIR(m)：是否是目录文件
*   S_ISSOCK(m)：是否是套接字

## 读取目录内容

&emsp;&emsp;程序可以调用 readdir 系列函数来读取目录内容：

```c
DIR *opendir(const char *name);
```

&emsp;&emsp;函数 opendir 以路径名为参数，返回指向`目录流`的指针。流是对条目有序列表的抽象，这里是指目录项的列表。

```c
struct dirent *readdir(DIR *dirp);
```

&emsp;&emsp;每次对 dirent 的调用返回的都是指向流 dirp 中下一个目录项的指针。如果没有更多目录项则返回 NULL。每个目录项都是一个结构：

```c
struct dirent {
    ino_t d_ino;       // inode number
    char  d_name[256]; // filename
}
```

&emsp;&emsp;使用 closedir 函数来关闭流并释放其所有资源。

```c
int closedir(DIR *dirp);
```

## 共享文件

&emsp;&emsp;内核使用三个相关的数据结构来表示打开的文件：

*   描述符表

&emsp;&emsp;每个进程都有它独立的描述符表，他的表项是由进程打开的文件描述符来索引的。每个打开的描述符表项指向文件表中的一个表项。

*   文件表

&emsp;&emsp;打开文件的集合是由一张文件表来表示的，所有的进程都共享这张表。每个文件表的表项组成包括当前的文件位置，引用计数，以及一个指向 v-node 表中的对应表项指针。关闭一个描述符会减少相应文件表项中的引用计数，除非它的引用计数为 0，否则内核不会删除这个文件表项。

*   v-node 表

&emsp;&emsp;同文件表一样，所有的进程共享这张 v-node 表。每个表项包含了 stat 结构中的大多数信息，包括 st_size 和 st_mode 等成员。

![](04.png)

&emsp;&emsp;多个描述符也可以通过不同的文件表表项来引用同一个文件。例如，用同一个 filename 调用 open 函数两次。关键思想在于：每个文件描述符都有它自己的文件位置，所以对不同描述符的读操作可以从文件的不同位置获取数据。

![](05.png)

&emsp;&emsp;在使用 fork 函数创建子进程时，子进程会获得父进程的文件描述符表副本，此后他们共享打开文件表集合，并且共享相同的文件位置：

![](06.png)

## I/O 重定向

&emsp;&emsp;shell 提供了 I/O 重定向操作符号，允许用户将磁盘文件和标准输入输出流关联起来。

```bash
ls > foo.txt
```

&emsp;&emsp;shell 加载和执行 ls 程序，将标准输出重定向到磁盘文件 foo.txt。

&emsp;&emsp;实现 I/O 重定向的一种方式是使用 dup2 函数：

```c
int dup2(int oldfd, int newfd);
```

&emsp;&emsp;dup2 函数复制描述符表项 oldfd 到 newfd，并覆盖表项 newfd 的内容。如果 newfd 已经打开，dup2 会在复制 oldfd 之前关闭 newfd。例如执行 dup2(4, 1)：

![](07.png)

&emsp;&emsp;在将磁盘文件重定向到标准输出后，文件 A 被关闭，其文件表表项的引用计数减少到 0，文件表表项和 v-node 表项都被删除。文件 B 的引用计数已经增加，此后，任何写到标准输出的内容都被重定向到文件 B。

## 标准 I/O

&emsp;&emsp;C 语言定义了一组高级 I/O 函数，称为标准 I/O 库。这个库由 libc 提供，包含了打开和关闭文件的函数 fopen 和 fclose。读写字节的函数 fread 和 fwrite。读写字符串的函数 fgets 和 fputs。以及复杂的格式化 I/O 函数 scanf 和 printf。

&emsp;&emsp;标准 I/O 将一个打开的文件模型化为一个流。简单来看，一个流就是一个指向 FILE 类型结构的指针。每个 C 开始时都有三个已打开的流 stdin，stdout 和 stderr。

```c
extern FILE *stdin;
extern FILE *stdout;
extern FILE *stderr;
```

&emsp;&emsp;类型 FILE 的流是对文件描述符和流缓冲区的抽象。流缓冲区的目的和 RIO 读缓冲区一样：使得开销较高的 Linux I/O 系统调用的次数尽量少一点。
