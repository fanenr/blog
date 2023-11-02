---
title: APUE 03 - 文件 I/O
date: 2023-09-20 20:22:55
updated: 2023-09-25 19:00:00
tags:
    - APUE
    - C
categories:
    - APUE
---

&emsp;&emsp;UNIX 系统中大多数文件 I/O 只需要用到 5 个函数：open, read, write, lseek 以及 close。

<!-- more -->

## 文件描述符

&emsp;&emsp;对于内核而言，所有打开的文件都通过文件描述符引用。文件描述符是一个非负整数，当打开一个现有文件或者创建一个新文件时，内核向进程返回一个文件描述符 (通常是系统调用 open 和 creat 返回)。当读写一个文件时，要将文件描述符作为参数传递给 read 和 write。

&emsp;&emsp;按照惯例，UNIX 系统 shell 把文件描述符 0, 1, 2 分别与进程的标准输入，标准输出，标准错误相关联。这是各种 shell 以及很多程序使用的惯例，与 UNIX 内核无关。虽然在 POSIX 应用程序中，幻数 0, 1, 2 已经被标准化，但是应当把它们替换成符号常量 STDIN_FILENO, STDOUT_FILENO, STDERR_FILENO。

&emsp;&emsp;文件描述符的变化范围是 0 ~ OPEN_MAX - 1。早期的 UNIX 系统实现采用的上限值是 19 (允许每个进程最多打开 20 个文件)。在现在大多数的 UNIX 系统上，文件描述符的变化范围几乎是无限的，它只受到系统配置的存储器总量，整型的字长以及系统管理员所配置的软限制和硬限制的约束。

## 函数 open 和 openat

&emsp;&emsp;调用 open 或 openat 函数可以打开或创建一个文件。

```c
#include <fcntl.h>

int open(const char *path, int oflag, ... /* mode_t mode */ );
int openat(int fd, const char *path, int oflag, ... /* mode_t mode */ );

/* Both return: file descriptor if OK, −1 on error */
```

&emsp;&emsp;最后一个参数 ... 是 ISO C 中的可变参数语法。对于 open 函数，仅当创建新文件时才使用它。

&emsp;&emsp;*path* 参数是要打开或创建文件的名字。*oflag* 参数可以用来说明此函数的多个选项，用下列一个或多个常量进行`或`运算构成 *oflag* 参数：

-   O_RDONLY：只读打开。
-   O_WRONLY：只写打开。
-   O_RDWR：读，写打开。

>   大多数实现将 O_RDONLY 定义为 0，O_WRONLY 定义为 1，O_RDWR 定义为 2，以与早期程序兼容。

-   O_EXEC：只执行打开。
-   O_SEARCH：只搜索打开。

>   O_SEARCH 常量的目的在于目录打开时验证它的搜索权限，对目录的文件描述符的后续操作就不需要再次检查对该目录的搜索权限。

&emsp;&emsp;在这 5 个常量中必须指定一个且只能指定一个 (Linux 中有区别)，下列常量则是可选的：

-   O_APPEND：每次写操作时都追加到文件的尾端。
-   O_CLOEXEC：把 FD_CLOEXEC 常量设置为文件描述符标志。
-   O_CREAT：若此文件不存在则创建它，使用此选项时，open 函数需同时说明第三个参数 *mode* (openat 中的第四个参数)，用 *mode* 指定该新文件的访问权限位。
-   O_DIRECTORY：如果 *path* 引用的不是目录，则出错。
-   O_EXCL：如果同时指定了 O_CREAT，而文件已存在，则出错。用此可以测试一个文件是否存在，如果不存在则创建文件，这使测试和创建两者成为一个原子操作。
-   O_NOCTTY：如果 *path* 引用的是终端设备，则不将该设备分配为此进程的控制终端。
-   O_NOFOLLOW：如果 *path* 引用的是一个符号链接，则出错。
-   O_NONBLOCK：如果 *path* 引用的是一个 FIFO，一个块特殊文件或者一个字符特殊文件，则此选项为文件的本次打开操作和后续的 I/O 操作设置为非阻塞方式。
-   O_SYNC：使每次 write 等待物理 I/O 操作完成，包括由该 write 操作引起的文件属性更新所需的 I/O。
-   O_TRUNC：如果此文件存在，而且为只写或读写成功打开，则将其长度截断为 0。
-   O_TTY_INIT：如果打开一个还未打开的终端设备，设置非标准 termios 参数值，使其符合 Single UNIX Specification。

&emsp;&emsp;下面两个标志也是可选的，它们是 Single UNIX Specification (以及 POSIX) 中同步输入和输出选项的一部分：

-   O_DSYNC：使每次 write 要等待物理 I/O 操作完成，但是如果该写操作并不影响读取刚写入的数据，则不需要等待文件属性被更新。

>   O_DSYNC 和 O_SYNC 标志的区别在于：仅当文件属性需要更新以反映文件数据变化 (例如更新文件大小以反映文件包含了更多数据) 时，O_DSYNC 标志才影响文件属性。而设置 O_SYNC 标志后，数据和属性总是同步更新。

-   O_RSYNC：是每一个以文件描述符作为参数进行的 read 操作等待，直至所有对文件同一部分挂起的写操作都完成。

&emsp;&emsp;由 open 和 openat 函数一定是最小的未用描述符值。这一点被某些应用用来在标准输入，标准输出，标准错误上打开新文件。例如：应用可以先关闭标准输出，然后打开另一个文件，执行打开操作前就能知道该文件一定会在描述符 1 上打开。

&emsp;&emsp;关于 openat 的 *fd* 参数，有 3 种可能性：

1.   *path* 参数指定的是绝对路径名，此时 *fd* 参数被忽略，openat 函数相当于 open 函数。
2.   *path* 参数指定的是相对路径名，*fd* 参数指出了相对路径名在文件系统中的开始地址。*fd* 参数是通过打开相对路径名所在的目录来获取的。
3.   *path* 参数指定的是相对路径名，*fd* 参数具有特殊值 AT_FDCWD，在这种情况下，路径名在当前工作目录中获取。openat 函数在操作上与 open 函数类似。

&emsp;&emsp;openat 函数希望解决两个问题：

-   让线程可以使用相对路径名打开其他目录中的文件，而不再只能打开当前工作目录中的文件。

-   可以避免 time-of-check-to-time-of-use (TOCTTOU) 错误。TOCTTOU 错误的基本思想是：如果有两个基于文件的函数调用，其中第二个调用依赖于第一个调用的结果，因为两个调用不是原子操作，在两个调用之间文件可能改变了，这样就造成第一个调用的结果不再有效，最终导致程序错误。

## 函数 creat

&emsp;&emsp;也可以调用  creat 函数创建一个新文件：

```c
#include <fcntl.h>

int creat(const char *path, mode_t mode);

/* Returns: file descriptor opened for write-only if OK, −1 on error */
```

&emsp;&emsp;此函数等效于：

```c
open(path, O_WRONLY | O_CREAT | O_TRUNC, mode)
```

&emsp;&emsp;creat 函数的缺点在于它以只写的方式打开所创建的文件，在提供 open 的新版本之前，如果要创建一个临时文件，并且要写读该文件，则必须调用 creat, close，然后再调用 open。现在可以直接调用 open 实现：

```c
open(path, O_RDWR | O_CREAT | O_TRUNC, mode)
```

## 函数 close

&emsp;&emsp;调用 close 函数关闭一个打开文件。

```c
#include <unistd.h>

int close(int fd);

/* Returns: 0 if OK, −1 on error */
```

&emsp;&emsp;关闭一个文件时还会释放该进程加在该文件上的所有记录锁。

&emsp;&emsp;当一个进程终止时，内核会自动关闭它所有的打开文件。

## 函数 lseek

&emsp;&emsp;每个打开文件都有一个与其相关量的当前文件偏移量 (current file offset)。它通常是一个非负整数，用以度量从文件开始处计算的字节数。通常，读写操作都从当前文件偏移量处开始，并使偏移量增加所读写的字节数。除非指定 O_APPEND 选项，否则系统默认的将偏移量设置为 0。

&emsp;&emsp;可以调用 lseek 函数显示的为一个打开文件设置偏移量。

```c
#include <unistd.h>

off_t lseek(int fd, off_t offset, int whence);

/* Returns: new file offset if OK, −1 on error */
```

&emsp;&emsp;参数 *offset* 的意义与参数 *whence* 的值有关：

-   若 *whence* 是 SEEK_SET，则将该文件的偏移量设置为距文件开始处 *offset* 个字节。
-   若 *whence* 是 SEE_CUR，则将该文件的偏移量设置为其当前值加 *offset*，*offset* 可为正或负。
-   若 *whence* 是 SEEK_END，则将该文件的偏移量设置为文件长度加 *offset*，*offset* 可为正或负。

&emsp;&emsp;若 lseek 成功执行，则返回新的文件偏移量，因此可以用下列方式确定打开文件的当前偏移量：

```c
off_t currpos;
currpos = lseek(fd, 0, SEEK_CUR);
```

&emsp;&emsp;这种方法也可以用来确定所涉及的文件是否允许设置偏移量，如果文件描述符指向一个管道，FIFO 或网络套接字，则 lseek 返回 -1，并将 errno 设置为 ESPIPE。

-----

&emsp;&emsp;通常，文件的当前偏移量应当是一个非负整数，某些设备可能允许负的偏移量。但是对于普通文件，其偏移量必须是一个非负值。因为偏移量可能是负值，所以在比较 lseek 的返回值时应该测试其是否为 -1，而不是检测其是否小于 0。lseek 仅将当前的文件偏移量记录在内核中，它并不引起任何 I/O 操作，该偏移量将应用与下一次的读或写操作中。

&emsp;&emsp;文件偏移量可以大于文件的当前长度，在这种情况下，对该文件的下一次写操作将加长该文件，并在文件中构成一个空洞，位于文件中但没有写过的字节都被读为 0。文件中的空洞并不要求在磁盘上占用存储区，具体的处理方式与文件系统的实现有关，对于新写的数据需要分配磁盘块，但是对于文件空洞则不需要分配磁盘块。

&emsp;&emsp;下面的程序用于创建一个具有空洞的文件：

```c
#include "apue.h"
#include <fcntl.h>

char buf1[] = "abcdefghi";
char buf2[] = "ABCDEFGHI";

int main(void)
{
    int fd;

    if ((fd = creat("file.hole", FILE_MODE)) < 0)
        err_sys("creat error");

    if (write(fd, buf1, 10) != 10)
        err_sys("buf1 write error");

    if (lseek(fd, 16384, SEEK_SET) == -1)
        err_sys("lseek error");

    if (write(fd, buf2, 10) != 10)
        err_sys("buf2 write error");
}
```

&emsp;&emsp;使用 ls(1) 和 od(1) 观察文件 file.hole 可以发现：文件中间的 30 个未写入字节都被读为 0，虽然文件大小是 16394，实际上该文件只占据 8 个磁盘块，而不带空洞的 file.nohole 文件要占据 20 个磁盘块。

&emsp;&emsp;因为 lseek 使用的偏移量是 off_t 类型，它允许实现根据特定的平台而选择合适大小的数据类型。现今大多数平台都支持两组接口以处理文件偏移量：一组是 32 位的文件偏移量，另一组使用 64 位。尽管可以实现 64 位文件偏移量，但是能否创建一个大于 2GB 的文件则依赖于底层文件系统的类型。

## 函数 read

&emsp;&emsp;调用 read 函数从打开文件中读数据。

```c
#include <unistd.h>

ssize_t read(int fd, void *buf, size_t nbytes);

/* Returns: number of bytes read, 0 if end of file, −1 on error */
```

&emsp;&emsp;如果 read 成功，则返回读到的字节数。如果以到达文件的尾端，则返回 0。

&emsp;&emsp;有多种情况可使实际读到的字节数少于要求读的字节数：

-   读普通文件时，在读到要求字节数之前已到达了文件尾端。
-   当从终端设备读时，通常一次最多读一行。
-   当从网络读时，网络中的缓冲机制可能造成返回值小于所要求读的字节数。
-   当从管道或 FIFO 读时，如果管道包含的字节数少于所需的数量，那么 read 将只返回实际可用的字节数。
-   当从某些面向记录的设备 (如磁带) 读时，一次最多返回一个记录。
-   当一信号造成中断，而已经读了部分数据时。

&emsp;&emsp;读操作从文件的当前偏移量开始，在成功返回之前，该偏移量将增加实际读到的字节数。POSIX 从几个方面对 read 函数的原型做了更改：

```c
/* classical */
int read(int fd, char *buf, unsigned nbytes);

/* POSIX */
ssize_t read(int fd, void *buf, size_t nbytes)
```

-   第二个参数由 char * 改为 void *，在 ISO C 中，类型 void * 用于表示通用指针。
-   返回值从 int 改为 ssize_t，类型 ssize_t 是一个带符号的整型，以保证能返回字节数，0 (EOF) 或 -1 (出错)。
-   第三个参数由无符号整型改为 size_t。

## 函数 write

&emsp;&emsp;调用 write 函数向打开文件写数据。

```c
#include <unistd.h>

ssize_t write(int fd, const void *buf, size_t nbytes);

/* Returns: number of bytes written if OK, −1 on error */
```

&emsp;&emsp;其返回值通常与参数 *nbytes* 相同，否则表示出错，write 出错的一个常见原因是磁盘已写满，或者超过了一个给定进程的文件长度限制。

&emsp;&emsp;对于普通文件，写操作从文件的当前偏移量处开始，如果在打开文件时指定了 O_APPEND 选项，则在每次写操作之前，将文件的当前偏移量设置在文件的当前结尾处，在一次写成功之后，偏移量增加写入的字节数。

## I/O 的效率

&emsp;&emsp;下面的程序只用 read 和 write 函数复制一个文件：

```c
#include "apue.h"

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

-   通过利用 shell 的 I/O 重定向功能，它可以不用打开输入文件和输出文件。
-   因为在进程终止时，UNIX 系统内核会关闭进程打开的所有文件描述符，所以此程序不手动关闭文件。
-   对 UNIX 系统内核而言，文本文件和二进制文件并无区别，所以此程序对两种文件都有效。

&emsp;&emsp;在 Linux ext4 文件系统下，磁盘块长度是 4096 字节。测试不同的 BUFFSIZE 下程序的执行效率：

![](01.png)

&emsp;&emsp;可以发现：系统 CPU 时间的几个最小值差不多出现在 BUFFSIZE 为 4096 及以后的位置，继续增加缓冲区长度对此时间几乎没有影响。

&emsp;&emsp;大多数文件系统为改善性能都采用某种`预读` (read ahead) 技术。当检测到正在进行顺序读时，系统就试图读入比应用所要求更多的数据，并假想应用很快就会读这些数据。预读的效果可以从图中看出，缓冲区为 32 字节时的时钟时间与拥有较大缓冲区的时钟时间几乎一样。

>   操作系统试图用高速缓存技术将相关文件放置在主存中，所以如果重复度量程序性能，那么后续运行该程序所得到的计时很可能好于第一次。其原因是：第一次运行使得文件进入高速缓存，后续各次运行一般从系统高速缓存访问文件，无需读，写磁盘。

## 文件共享

&emsp;&emsp;UNIX 系统支持在不同的进程间共享打开文件。内核使用 3 种数据结构来表示打开文件，它们之间的关系决定了在文件共享方面一个进程对另一个进程可能产生的影响：

1.   在每个进程的进程表中都有一个记录项，该记录项中包含一张打开文件描述符表，可以将其看作一个矢量，每个描述符占用一项，与每个文件描述符关联的内容有：

-   文件描述符标志 (close_on_exec)。
-   指向一个文件表表项的指针。

2.   内核为所有打开文件维持一张文件表，一个文件表表项包含：

-   文件状态标志 (读，写，添写，同步和非阻塞等)。
-   当前文件偏移量。
-   指向该文件 v-node 表项的指针。

3.   每个打开文件 (或设备) 都有一个 v-node 结构，v-node 包含了文件类型和对此文件进行操作各种函数的指针。对于大多数文件，v-node 还包含了该文件的 i-node (索引节点)。这些信息是在打开文件时从磁盘读入内存的，所以文件的所有相关信息都是随时可用的。例如 i-node 包含了文件所有者，文件长度，指向文件实际数据块在磁盘上所在位置的指针等。

>   Linux 没有使用 v-node 表项，而是使用了通用 i-node 结构 (一个与文件系统相关的 i-node 和一个与文件系统无关的 i-node)。虽然二者实现有所不同，但是概念上 v-node 和 i-node 是一样的，它们都指向文件系统特有的 i-node 结构。

![](02.png)

&emsp;&emsp;如果两个进程各自打开了同一文件，关系如下：

![](03.png)

&emsp;&emsp;打开该文件的每个进程都获得各自的一个文件表项，但是对一个给定的文件，只有一个 v-node 表项。每个进程都有各自的文件表项，是为了让每个进程都有自己对该文件的当前偏移量。

-   完成 write 后，要在文件表项中的当前文件偏移量增加所写入的字节数。如果这导致当前文件偏移量超出了当前文件长度，则将 i-node 表项中的当前文件长度设置为当前文件偏移量。
-   如果用 O_APPEND 标志打开一个文件，则相应标志也被设置到文件表项中的文件状态标志中。每次对具有这种标志的文件执行写操作时，文件表项中的当前文件偏移量首先会被设置为 i-node 表项中的文件长度。
-   若一个文件使用 lseek 定位到文件尾端，则文件表项中的当前文件偏移量被设置为 i-node 表项中的当前文件长度 (这与使用 O_APPEND 标志打开文件不同)。
-   lseek 函数只修改文件表项中的当前文件偏移量，不进行任何 I/O 操作。

-----

&emsp;&emsp;可能有多个文件描述符指向同一文件表项，如 dup 函数和 fork 函数。在子进程创建后，父子进程中各自的打开文件描述符共享同一个文件表项。

&emsp;&emsp;文件描述符标志只作用于一个进程的一个描述符，而文件状态标志则应用于指向给定文件表项的任何进程中的所有描述符，可以使用 fcntl 函数获取和修改这两个标志。

&emsp;&emsp;多个进程读取同一个文件是没问题的，因为每个进程都有它自己的文件表项，其中也有它自己的当前文件偏移量。但是，当多个进程写同一个文件时，结果可能混乱。

## 原子操作

&emsp;&emsp;一般而言，`原子操作` (atomic operation) 指的是由多步组成的一个操作。如果该操作原子的执行，则要么执行完所有的步骤，要么一步也不执行，不可能只执行所有步骤的一个子集。

1.   追加到一个文件

&emsp;&emsp;考虑一个进程，它要将数据追加到一个文件尾端。早期的 UNIX 系统并不支持 open 的 O_APPEND 选项，所以程序被编写成下列形式：

```c
if (lseek(fd, 0L, 2) < 0)
    err_sys("lseek error");

if (write(fd, buf, 100) != 100)
    err_sys("write error");
```

&emsp;&emsp;对于单个进程而言，它能正常工作，但是若有进程同时使用这种方式将数据追加到同一文件则会产生问题：简单来说，如果一个进程刚刚将文件的当前偏移量设置到文件末尾，但是此时内核切换到另一个进程，另一个进程完成追加操作。但是等到调度回到原来进程时，此时文件长度已经改变了，但它却不知道，只能覆盖刚刚另一个进程写入的信息。

&emsp;&emsp;问题的逻辑出现在**先定位，然后写**，它使用了两个分开的函数调用。解决问题的方法是使这两个操作对于其他进程而言成为一个原子操作。任何要求多于一个函数调用的操作都不是原子操作，因为在两个函数调用之间，内核有可能会临时挂起进程。

&emsp;&emsp;UNIX 通过提供 O_APPEND 打开文件标志解决这个问题，这样做使得内核在每次写操作之前，都将进程的的当前偏移量设置到该文件的尾端，而不需要在写之前调用 lseek。

2.   函数 pread 和 pwrite

&emsp;&emsp;Single UNIX Specification 包括了 XSI 扩展，该扩展允许原子性的定位并执行 I/O。

```c
#include <unistd.h>

ssize_t pread(int fd, void *buf, size_t nbytes, off_t offset);
/* Returns: number of bytes read, 0 if end of file, −1 on error */

ssize_t pwrite(int fd, const void *buf, size_t nbytes, off_t offset);
/* Returns: number of bytes written if OK, −1 on error */
```

&emsp;&emsp;调用 pread 相当于调用 lseek 后调用 read，但是 pread 又与这种顺序调用有重要区别：

-   调用 pread 时，无法中断其定位和读操作。
-   不更新当前文件偏移量。

&emsp;&emsp;调用 pwrite 相当于调用 lseek 后调用 write，区别类似以上。

3.   创建一个文件

&emsp;&emsp;之前说明 open 函数的选项时提到过 O_CREAT 和 O_EXCL 选项，其将判断文件不存在和创建文件合并为一个原子操作。如果没有这个原子操作，那么可能会编写下面的代码：

```c
if ((fd = open(path, O_WRONLY)) < 0) {
    if (errno == ENOENT) {
        if ((fd = creat(path, mode)) < 0)
            err_sys("creat error");
    } else {
        err_sys("open error");
    }
}
```

&emsp;&emsp;如果在 open 和 creat 之间，另一个进程创建了该文件，就会出现问题。如果另一进程在此空隙间创建了文件，并写入了数据，那么原进程中的 creat 将擦去数据。

## 函数 dup 和 dup2

&emsp;&emsp;下面两个函数都可用来复制一个现有的描述符：

```c
#include <unistd.h>

int dup(int fd);
int dup2(int fd, int fd2);

/* Both return: new file descriptor if OK, −1 on error */
```

&emsp;&emsp;由 dup 返回的新文件描述符一定是当前可用文件描述符中的最小数值。对于 dup2，可以用 *fd2* 参数指定新描述符的值，如果 *fd2* 已经打开，则先将其关闭。如果 *fd* 等于 *fd2*，则 dup2 返回 *fd2*，而不关闭它，否则，*fd2* 的 FD_CLOEXEC 文件描述符标志被清除，这样的话，*fd2* 在进程调用 exec 后还是打开状态。

&emsp;&emsp;这两个函数返回的新文件描述符与参数 *fd* 共享同一个文件表项：他们将共享同一文件状态标志 (可读，可写，读写，追加等)，以及同一个当前文件偏移量。

![](04.png)

&emsp;&emsp;每个文件描述符都有自己的一套文件描述符标志，新文件描述符的执行时关闭 (close-on-exec) 标志总是被 dup 函数清除，当 fd 和 *fd2* 不等时被 dup2 清除。

-----

&emsp;&emsp;复制一个描述符的另一个方法是使用 fcntl 函数：

```c
dup(fd);
/* like */
fcntl(fd, F_DUPFD, 0);

dup2(fd, fd2);
/* like */
close(fd);
fcntl(fd, F_DUPFD, fd2);
```

&emsp;&emsp;后一种情况中，dup2 并不完全等同于 close 加上 fcntl。区别如下：

-   dup2 是一个原子操作，而 close 和 fcntl 包括两个函数调用。
-   dup2 和 fcntl 有一些不同的 errno。

## 函数 sync, fsync 和 fdatasync

&emsp;&emsp;传统的 UNIX 系统实现在内核中设有缓冲区高速缓存或页高速缓存，大多数磁盘 I/O 都通过缓冲区进行。当向文件写入数据时，内核通常先将数据复制到缓冲区中，然后排入队列，晚些时候再写入磁盘。这种方式被称为`延迟写` (delayed write) 或`写回`。

&emsp;&emsp;通常，当内核需要重用缓冲区来存放其他磁盘块数据时，它会把所有延迟写数据块写入磁盘。为了保证磁盘上的实际文件系统与缓冲区中的内容一致性，UNIX 系统提供了 sync，fsync 和 fdatasync 三个函数：

```c
#include <unistd.h>

int fsync(int fd);
int fdatasync(int fd);

/* Returns: 0 if OK, −1 on error */

void sync(void);
```

&emsp;&emsp;sync 只是将所有修改过的块缓冲区排入写队列，然后就返回，**它并不等待实际写磁盘操作结束。**

&emsp;&emsp;通常，称为 update 的系统守护进程周期性的调用 (一般间隔 30 秒) sync 函数。这就保证了定期冲洗 (flush) 内核的块缓冲区。命令 sync(1) 也调用 sync 函数。

&emsp;&emsp;fsync 函数只对由文件描述符指定的一个文件起作用，并且等待写磁盘操作结束才返回。

&emsp;&emsp;fdatasync 函数类似 fsync，但它只影响文件的数据部分。而除数据外，fsync 还会同步更新文件的属性。

## 函数 fcntl

&emsp;&emsp;fcntl 函数可以改变已经打开文件的属性。

```c
#include <fcntl.h>

int fcntl(int fd, int cmd, ... /* int arg */ );

/* Returns: depends on cmd if OK (see following), −1 on error */
```

&emsp;&emsp;fcntl 函数有以下 5 种功能：

1.   复制一个已有的描述符 (*cmd* = F_DUPFD 或 F_DUPFD_CLOEXEC)。
2.   获取/设置文件描述符标志 (*cmd* = F_GETFD 或 F_SETFD)。
3.   获取/设置文件状态标志 (*cmd* = F_GETFL 或 F_SETFL)。
4.   获取/设置异步 I/O 所有权 (*cmd* = F_GETOWN 或 F_SETOWN)。
5.   获取/设置记录锁 (*cmd* = F_GETLK, F_SETLK, 或 F_SETLKW)。

&emsp;&emsp;前 8 个标志的含义：

-   F_DUPFD：复制文件描述符 *fd*，新文件描述符作为函数返回值返回。它是尚未打开描述符中大于或等于第三个参数中的最小值。新描述符与 *fd* 共享同一文件表项，但是新文件描述符有自己的文件描述符标志，它的 FD_CLOEXEC 标志被清除。
-   F_DUPFD_CLOEXEC：也是复制文件描述符，但是此标志会保留新文件描述符的 FD_CLOEXEC 标志。
-   F_GETFD：返回对应 *fd* 的文件描述符标志，当前只定义了一个文件描述符标志 FD_CLOEXEC。
-   F_SETFD：将对应 *fd* 的文件描述符标志设置为第三个参数。
-   F_GETFL：返回对应 *fd* 的文件状态标志，有以下状态：

![](05.png)

-   F_SETFL：将文件标志设置为第三个参数，可以更改的标志有：O_APPEDN，O_NONBLOCK，O_SYNC，O_DSYNC，O_RSYNC，O_FSYNC 和 O_ASYNC。

-   F_GETOWN：获取当前接收 SIGIO 和 SIGURG 信号的进程 ID 或进程组 ID。

-   F_SETOWN：设置接收 SIGIO 和 SIGURG 信号的进程 ID 或进程组 ID。正的 *arg* 指定一个进程 ID，负的 *arg* 指定一个其绝对值的进程组 ID。

&emsp;&emsp;fcntl 函数的返回值与命令 *cmd* 有关，如果出错，所有命令都返回 -1，如果成功则返回某个其他值。

-----

&emsp;&emsp;下面的程序将传给其第一个参数当作 *fd* 并打印其文件标志：

```c
    #include "apue.h"
    #include <fcntl.h>
    
    int main(int argc, char **argv)
    {
        int val;
    
        if (argc != 2)
            err_quit("usage: %s <description#>", argv[0]);
    
        if ((val = fcntl(atoi(argv[1]), F_GETFL, 0)) < 0)
            err_sys("fcntl error for fd %d", atoi(argv[1]));
    
        switch (val & O_ACCMODE) {
        case O_RDONLY:
            printf("read only");
            break;
        case O_WRONLY:
            printf("write only");
            break;
        case O_RDWR:
            printf("read write");
            break;
        default:
            err_dump("unknown access mode");
        }
    
        if (val & O_APPEND)
            printf(", append");
    
        if (val & O_NONBLOCK)
            printf(", nonblocking");
    
        if (val & O_SYNC)
            printf(", synchronous writes");
    
    #if !defined(_POSIX_C_SOURCE) && defined(O_FSYNC) && (O_FSYNC != O_SYNC)
        if (val & O_FSYNC)
            printf(", synchronous writes");
    #endif
    
        putchar('\n');
    }
```

&emsp;&emsp;注意检测 5 个访问模式的方式：使用了宏 O_ACCMODE，这是因为 5 个访问模式并不各占一位。其次，程序使用了测试宏 _POSIX_C_SOURCE 条件编译了 O_FSYNC 的检验。最后，在 shell 中测试这个程序：

```bash
    $ ./a.out 0 < /dev/tty
    read only
    $ ./a.out 1 > temp.foo
    $ cat temp.foo
    write only
    $ ./a.out 2 2>>temp.foo
    write only, append
    $ ./a.out 5 5<>temp.foo
    read write
```

&emsp;&emsp;第 6 行中 2>>temp.foo 语法的含义是：在程序中以 O_APPEND 的标志打开文件 temp.foo，并将文件描述符 2 重定向到打开文件 temp.foo。第 8 行中 5<>temp.foo 的含义类似，它是在文件描述符 5 上以读写标志打开文件 temp.foo。

-----

&emsp;&emsp;修改文件描述符略有不同，首先必须要获取现在的标志值，然后按照期望值修改它，最后设置新标志值，不能只执行 F_SETFL 或 F_GETFL 命令，这样会关闭以前设置的标志值。

```c
void set_fl(int fd, int flags)
{
    int val;

    if ((val = fcntl(fd, F_GETFL, 0)) < 0)
        err_sys("fcntl F_GETFL error");

    val |= flags;

    if (fcntl(fd, F_SETFL, val) < 0)
        err_sys("fcntl F_SETFL error");
}
```

&emsp;&emsp;如果将第 8 行中改为 val &= ~flags，就构成了另一个函数 clr_fl，它的功能正好和 set_fl 相反。

## 函数 ioctl

&emsp;&emsp;ioctl 函数是 Single UNIX Specification 标准中的一个扩展部分，以便处理 STREAMS 设备：

```c
#include <unistd.h>     /* System V */
#include <sys/ioctl.h>  /* BSD and Linux */

int ioctl(int fd, int request, ...);

/* Returns: −1 on error, something else if OK */
```

&emsp;&emsp;每个驱动设备可以定义它自己专用的 ioctl 命令，系统为不同种类的设备提供通用的 ioctl 命令，下面是 FreeBSD 支持的通用 ioctl 命令的一些类别：

![](06.png)

&emsp;&emsp;磁带操作允许在磁带上写一个文件结束标志，倒带，越过指定个数的文件或记录等。但是这些操作很难用 read, write, lseek 来表示，所以，对这些设备进行操作最容易的方法就是使用 ioctl。

## /dev/fd

&emsp;&emsp;较新的系统都提供名为 /dev/fd 的目录，其目录项是名为 0, 1, 2 ... 的文件，打开文件 /dev/fd/*n* 等效于复制描述符 n (假定文件描述符 n 是打开的)。

```c
fd = open("/dev/fd/0", mode);
/* like */
fd = dup(0);
```

&emsp;&emsp;大多数系统忽略它指定的 mode，而另一些系统要求 mode 必须是所引用文件初始打开时的一个子集。即使系统忽略打开模式，而且下面的调用是成功的：

```c
fd = open("/dev/fd/0", O_RDWR);
```

&emsp;&emsp;但是仍然不能对 fd 进行写操作，因为标准输入无法写入。

>   Linux 实现中的 /dev/fd 是个例外。它把文件描述符映射成指向底层物理文件的符号链接。