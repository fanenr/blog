---
title: APUE 05 - 标准 I/O 库
date: 2023-09-30 22:04:43
updated: 2023-10-02 19:00:00
tags:
    - APUE
    - C
categories:
    - APUE
---

&emsp;&emsp;标准 I/O 库处理了很多细节，如缓冲区分配，以优化的块长度执行 I/O 等。

<!-- more -->

## 流和 FILE 对象

&emsp;&emsp;之前的所有 I/O 函数都围绕文件描述符，打开或创建一个文件都会获得一个文件描述符，然后将该文件描述符用于后续的 I/O 操作。而对于标准 I/O 库，其操作是围绕`流` (stream) 进行的。当用标准 I/O 库打开或创建一个文件时，就会获得一个与该文件相关联的流。

&emsp;&emsp;对于 ASCII 字符集，一个字符用一个字节表示。对于国际字符集 Unicode，一个字符可用多个字节表示。标准 I/O 文件流可用于单字节或多字节字符集。`流的定向` (stream's orientation) 决定了所读，写的字符是单字节的还是多字节的。

&emsp;&emsp;当一个流最初被创建时，它没有定向。若在未定向的流上使用一个多字节 I/O 函数，则该流被定向为多字节。若在未定向的流上使用一个单字节 I/O 函数，则该流被定向为单字节。只有两个函数可以改变流的定向：freopen 函数用于清除一个流的定向，fwide 函数用于设置流的定向：

```c
#include <stdio.h>
#include <wchar.h>

int fwide(FILE *fp, int mode);

/* Returns: positive if stream is wide oriented, negative if stream is byte oriented, or 0 if stream has no orientation */
```

&emsp;&emsp;根据 *mode* 参数的不同值，fwide 函数执行不同的工作：

-   如果 *mode* 为负值，fwide 将试图指定流为字节定向。
-   如果 *mode* 为正值，fwide 将试图指定流为宽定向。
-   如果 *mode* 为 0，fwide 将不指定流的定向，当返回标识该流定向的值。

&emsp;&emsp;注意：如果一个流已经有了定向，那么 fwide 将不会修改其定向，而是直接返回当前的定向。并且，fwide 无出错返回。唯一可靠的是，在调用 fwide 前先清除 errno，在调用 fwide 后检查 errno 的值。

&emsp;&emsp;在打开一个流时，标准 I/O 函数 fopen 返回一个指向 FILE 对象的指针。该对象通常是一个结构，它包含了标准 I/O 库为管理该流所需要的所有信息，包括用于实际 I/O 的文件描述符，指向用于该流缓冲区的指针，缓冲区的长度，当前在缓冲区的字符数以及出错标志等。

&emsp;&emsp;为了引用一个流，需要将 FILE 指针 (文件指针) 作为参数传递给每个标准 I/O 函数。

## 标准输入，输出和错误

&emsp;&emsp;UNXI shell 为每个新进程都打开 3 个预定义的流：标准输入，标准输出和标准错误。这些流引用的文件和之前提到的文件描述符 STDIN_FILENO，STDOUT_FILENO 和 STDERR_FILENO 所引用的文件相同。

&emsp;&emsp;这 3 个标准 I/O 流通过预定义文件指针 stdin，stdout 和 stderr 加以引用。

## 缓冲

&emsp;&emsp;标准 I/O 库提供缓冲的目的是尽可能的减少使用 read 和 write 的调用次数。它也对每个 I/O 流自动地进行缓冲管理，从而避免应用程序需要考虑缓冲问题而带来的麻烦。

&emsp;&emsp;标准 I/O 提供以下 3 种类型的缓冲：

1.   全缓冲

&emsp;&emsp;这种情况下，在填满标准 I/O 缓冲区后才执行实际的 I/O 操作。对于驻留在磁盘上的文件通常是由标准 I/O 库实施全缓冲的。在一个流上执行第一次 I/O 操作时，相关的标准 I/O 函数通常调用 malloc 获得需使用的缓冲区。

&emsp;&emsp;术语`冲洗` (flush) 说明标准 I/O 缓冲区的写操作。缓冲区可由标准 I/O 例程自动的冲洗 (如填满一个缓冲区时)，或者是可以调用函数 fflush 手动的冲洗一个流。

2.   行缓冲

&emsp;&emsp;这种情况下，当在输入和输出中遇到换行符时，标准 I/O 库执行 I/O 操作。这允许一次只输出一个字符，但只有在写了一行之后才进行实际 I/O 操作。当涉及一个终端时，通常使用行缓冲。

&emsp;&emsp;行缓冲有两个限制：第一，因为标准 I/O 库用来收集每一行的缓冲区的长度是固定的，所以只要填满了缓冲区，那么即使还没有写一个换行符，也进行 I/O 操作。第二，任何时候只要通过标准 I/O 库要求从一个不带缓冲的流，或者一个行缓冲的流中得到输入数据，那么就会冲洗所有行缓冲输出流。原因是：所需要的数据可能已在该缓冲区中。

3.   无缓冲

&emsp;&emsp;标准 I/O 库不对字符进行缓冲存储。标准错误流 stderr 通常是不带缓冲的，这就使得出错信息可以尽快的显示出来，而不管它们是否含有一个换行符。

-----

&emsp;&emsp;ISO C 要求下列缓冲特征：

-   当且仅当标准输入和标准输出并不指向交互式设备时，它们才是全缓冲的。
-   标准错误绝不会是全缓冲的。

&emsp;&emsp;但是，标准并未说明如果标准输入和标准输出指向交互设备时，它们是不带缓冲的还是行缓冲的。以及标准错误是不带缓冲的还是行缓冲的。很多系统默认使用下列类型的缓冲：

-   标准错误是不带缓冲的。
-   若是指向终端设备，则是行缓冲的，否则是全缓冲的。

&emsp;&emsp;对任何一个给定的流，如果不喜欢系统默认设置，可以调用下列函数更改缓冲类型：

```c
#include <stdio.h>

void setbuf(FILE *restrict fp, char *restrict buf);
int setvbuf(FILE *restrict fp, char *restrict buf, int mode, size_t size);

/* Returns: 0 if OK, nonzero on error */
```

&emsp;&emsp;可以使用 setbuf 函数打开或关闭缓冲机制。为了带缓冲进行 I/O，参数 *buf* 必须指向一个长度为 BUFSIZ 的缓冲区 (该常量定义在 <stdio.h> 中)。通常在此之后，该流就是全缓冲的，但如果该流关联一个终端设备，某些系统也可将其设置为行缓冲。将 *buf* 设置为 NULL 则可以关闭缓冲。

&emsp;&emsp;使用 setvbuf，可以通过指定 *mode* 的值，精确的说明所需的缓冲类型*：

-   _IOFBF：全缓冲。
-   _IOLBF：行缓冲。
-   _IONBF：无缓冲。

&emsp;&emsp;如果指定一个流不带缓冲，则忽略其 *buf* 和 *size* 参数。如果指定全缓冲或行缓冲，则通过 *buf* 和 *size* 可以选择性的指定一个缓冲区及其长度。如果该流是带缓冲的，但是 *buf* 是 NULL，则标准 I/O 将自动的为该流分配适当长度的缓冲区 (由常量 BUFSIZ 所指定长度的缓冲区)。

>   GNU C 函数库使用 stat 结构中的成员 st_blksize 所指定的值决定最佳 I/O 缓冲区长度。

![](01.png)

&emsp;&emsp;如果在一个函数内使用自动变量 (栈上变量) 作为流的标准 I/O 缓冲区，则从该函数返回前，必须关闭该流。

&emsp;&emsp;任何时候，都可以强制冲洗一个流：

```c
#include <stdio.h>

int fflush(FILE *fp);

/* Returns: 0 if OK, EOF on error */
```

&emsp;&emsp;fflush 使该流所有未写的数据都被传送至内核。特殊的，如果 *fp* 是 NULL，将导致所有的输出流被冲洗。

## 打开流

&emsp;&emsp;下列 3 个函数打开一个标准 I/O 流：

```c
#include <stdio.h>

FILE *fopen(const char *restrict pathname, const char *restrict type);
FILE *freopen(const char *restrict pathname, const char *restrict type, FILE *restrict fp);
FILE *fdopen(int fd, const char *type);

/* All three return: file pointer if OK, NULL on error */
```

&emsp;&emsp;它们的区别如下：

1.   fopen 函数打开路径名为 *pathname* 的指定文件。
2.   freopen 函数在一个指定流上打开一个指定文件，如果该流已经打开，则先关闭该流。如果该流已经定向，则使用 freopen 函数清除该定向。此函数一般用于将一个指定文件打开为一个预定义的流：3 个标准流。
3.   fdopen 函数取一个已有的文件描述符，并使一个标准 I/O 流与该描述符结合。此函数常用于由创建管道和网络通信通道函数返回的描述符。

&emsp;&emsp;*type* 参数指定对该 I/O 流的读写方式，ISO C 规定 *type* 参数可以有 15 种值：

![](02.png)

&emsp;&emsp;字符 b 作为 *type* 的一部分，使得标准 I/O 系统可以区分文本文件和二进制文件，但是 UNIX 并不对这两种文件进行区分，所以 b 字符在 UNIX 上没有作用。

&emsp;&emsp;打开一个流的 6 种方式：

![](03.png)

&emsp;&emsp;对于 fdopen 函数，因为它基于一个已存在的文件描述符，并且该文件总是存在的。所以，fdopen 不能截断它为写而打开的任一文件。并且，标准 I/O 追加方式也不能用于创建该文件。

&emsp;&emsp;当用追加写类型打开一个文件后，每次写都将数据写到文件尾端。如果有多个进程用标准 I/O 追加写同一个文件，那么来自每个进程的数据都将正确的写到文件中。

&emsp;&emsp;当以读写方式打开一个文件时 (*type* 中的 + 号)，具有以下限制：

-   如果中间没有 fflush，fseek，fsetpos 或 rewind，则在输出的后面不能直接跟随输入。
-   如果中间没有 fseek，fsetpos 或 rewind，或者一个输入操作没有到达文件尾端，则在输入操作之后不能直接跟随输出。

&emsp;&emsp;在指定 w 或 a 类型创建一个新文件时，是无法说明该文件的访问权限位的。POSIX 要求实现使用如下的权限位集来创建文件，然后，可以通过调整进程的 umask 值来限制这些权限：

```c
S_IRUSR | S_IWUSR | S_IRGRP | S_IWGRP | S_IROTH | S_IWOTH
```

&emsp;&emsp;除非流引用终端设备，否则按系统默认，流被打开时是全缓冲的。若流引用终端设备，则该流是行缓冲的。要改变一个流的缓冲类型，可以使用 setbuf 和 setvbuf 函数。

&emsp;&emsp;最后，可以调用函数 fclose 函数关闭一个打开的流：

```c
#include <stdio.h>

int fclose(FILE *fp);

/* Returns: 0 if OK, EOF on error */
```

&emsp;&emsp;在该文件被关闭之前，冲洗缓冲区的输出数据。缓冲区中的任何输入数据将被丢弃。如果标准 I/O 库已为该流自动分配了一个缓冲区，则释放此缓冲区。

&emsp;&emsp;当一个进程正常终止时 (调用 exit 或从 main 函数返回)，则所有带有未写缓冲数据的标准 I/O 流都将被冲洗，所有打开的标准 I/O 流都被关闭。

## 读和写流

&emsp;&emsp;一旦打开了一个流，就有 3 种不同类型的非格式化 I/O 方式对其进行读写：

1.   每次一个字符的 I/O。一次读或写一个字符，如果流是带缓冲的，则标准 I/O 函数处理所有缓冲。
2.   每次一行的 I/O。使用 fgets 和 fputs 一次读或写一行。
3.   直接 I/O，又叫二进制 I/O。函数 fread 和 fwrite 支持这种类型的 I/O，每次 I/O 操作读或写某个数量的对象，而每个对象具有指定的长度。这两个函数常用于二进制文件的读写。

-----

1.   输入函数

&emsp;&emsp;以下 3 个函数可用于一次读一个字符：

```c
#include <stdio.h>

int getc(FILE *fp);
int fgetc(FILE *fp);
int getchar(void);

/* All three return: next character if OK, EOF on end of file or error */
```

&emsp;&emsp;函数 getchar 等同于 getc(stdin)。前两个函数的区别是，getc 可以被实现为宏，而 fgetc 不能实现为宏：

1.   getc 的参数不应当是具有副作用的表达式，因为它可能会被计算多次。
2.   fgetc 一定是一个函数，所以可以取它的地址。
3.   调用 fgetc 所需的时间可能比 getc 长，这是函数和宏的区别。

&emsp;&emsp;这 3 个函数的返回值都是 int，这是为了能返回 EOF 标志，EOF 通常被定义为 -1。并且，无论是出错还是到达文件尾端，他们都返回 -1，要区分情况，必须调用 ferror 或 feof：

```c
#include <stdio.h>

int ferror(FILE *fp);
int feof(FILE *fp);

/* Both return: nonzero (true) if condition is true, 0 (false) otherwise */

void clearerr(FILE *fp);
```

&emsp;&emsp;在大多数实现中，每个流在 FILE 对象中都有如下两个标志，调用 clearerr 可以清除这两个标志。

-   出错标志。
-   文件结束标志。

&emsp;&emsp;从流中读取数据后，可以调用 ungetc 将字符再压送回流中：

```c
#include <stdio.h>

int ungetc(int c, FILE *fp);

/* Returns: c if OK, EOF on error */
```

&emsp;&emsp;压送回的字符以后又可以从流中读出，但读出的字符顺序与压送回的顺序相反。回送的字符不必要是上一次读到的字符，但是不能回送 EOF。但是，当已经到达文件尾端时，仍然可以回送一个字符，并且此时 ungetc 还会清除流的 EOF 标志。

>   用 ungetc 压送回字符时，并没有将它写到底层的文件或设备上，只是将他们写回标准 I/O 库的流缓冲区中。

2.   输出函数

&emsp;&emsp;上面 3 个输入函数分别有一个对应的输出函数：

```c
#include <stdio.h>

int putc(int c, FILE *fp);
int fputc(int c, FILE *fp);
int putchar(int c);

/* All three return: c if OK, EOF on error */
```

&emsp;&emsp;putchar 等同于 putc(c, stdout)，putc 可能被实现为 宏，而 fputc 必须是函数。

## 每次一行 I/O

&emsp;&emsp;下面两个函数提供每次输入一行的功能：

```c
#include <stdio.h>

char *fgets(char *restrict buf, int n, FILE *restrict fp);
char *gets(char *buf);

/* Both return: buf if OK, NULL on end of file or error */
```

&emsp;&emsp;这两个函数都指定了缓冲区的地址，用于保存读入的行。gets 从标准输入读，fgets 则从指定的流读。

&emsp;&emsp;对于 fgets，必须指定缓冲区长度 *n*。fgets 会一直读取字符，直到遇到换行符，NULL 字符或者读到 EOF。但是读取不超过 n-1 个字符，读入的字符被送入缓冲区。fgets 总是在读取到的数据末尾添加 NULL 字符，如果该行字符 (包括换行符) 数超过 n-1，则 fgets 只返回一个不完整的行。

&emsp;&emsp;函数 gets 已经不推荐使用，问题在于调用者在使用时不能指定缓冲区的长度，这样可能造成缓冲区溢出。并且，gets 与 fgets 不同的是它会删除换行符。

&emsp;&emsp;fputs 和 puts 提供每次输出一行的功能：

```c
#include <stdio.h>

int fputs(const char *restrict str, FILE *restrict fp);
int puts(const char *str);

/* Both return: non-negative value if OK, EOF on error */
```

&emsp;&emsp;二者都写出一个以 NULL 字符结尾的字符串，区别在于：puts 会在字符串后自动添加一个换行符，而 fputs 则不会。

## 标准 I/O 的效率

&emsp;&emsp;使用 getc 和 putc 将标准输入复制到标准输出的例子：

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

&emsp;&emsp;可以使用 fgetc 和 fputc 改写，它们一定是函数而不是宏。下面是读写为行 I/O 的版本：

```c
#include "apue.h"

int main(void)
{
    char buf[MAXLINE];
    while (fgets(buf, MAXLINE, stdin) != NULL)
        if (fputs(buf, stdout) == EOF)
            err_sys("output error");
    if (ferror(stdin))
        err_sys("input error");
}
```

&emsp;&emsp;这 3 个程序对同一文件进行操作的时间数据：

![](04.png)

&emsp;&emsp;可以发现，这 3 个标准 I/O 版本中的每一个，其用户 CPU 时间都大于最佳 read 版本。每次读一个字节的版本要执行 1 亿次循环，每次读一行的版本要执行 3 144 984 次循环。而 read 版本只有 25 244 次。因为系统 CPU 时间几乎相同，所以用户 CPU 时间以及等待 I/O 结束所消耗的时间是时钟时间差距的主要原因。

&emsp;&emsp;系统 CPU 时间几乎相同，这是因为这些程序对内核提出的读，写请求数基本相同。

&emsp;&emsp;使用每次一行 I/O 版本的速度大约是每次一个字节版本速度的两倍多。在这个例子中，原因在于每次一行 I/O 使用 memcpy(3) 实现，并且 memcpy 函数使用汇编而非 C 语言编写。

&emsp;&emsp;对比最后两行，fgetc 版本比 BUFFSIZE = 1 的版本要快得多。二者都使用了大约 2 亿次函数调用，但是区别在于：read 版本引起了 2 亿次系统调用，而 fgetc 版本只引起 25 244 次系统调用。而系统调用比普通函数调用要花费更长时间。

## 二进制 I/O

&emsp;&emsp;如果要进行二进制 I/O 操作，并且要求一次读或写一个完整结构，那么单字节 I/O 和行 I/O 都不太适用：fgetc 和 fputc 每次只能处理一个字符，需要在循环中反复调用。fgets 和 fputs 适用于行操作，并且它们对于 NULL 字符和换行符有特殊处理，而结构中可能包含这些字符。下面两个函数更适合二进制 I/O 操作：

```c
#include <stdio.h>

size_t fread(void *restrict ptr, size_t size, size_t nobj, FILE *restrict fp);
size_t fwrite(const void *restrict ptr, size_t size, size_t nobj, FILE *restrict fp);

/* Both return: number of objects read or written */
```

&emsp;&emsp;这些函数有两种常见的写法：

1.   读或写一个二进制数组

```c
float data[10];

if (fwrite(&data[2], sizeof(float), 4, fp) != 4)
    err_sys("fwrite error");
```

&emsp;&emsp;这将一个浮点数组索引为 2~5 的元素写到文件上，*size* 为单个数组元素的大小，*nobj* 为要写的元素个数。

2.   读或写一个结构

```c
struct {
    short count;
    long  total;
    char  name[NAMESIZE];
} item;

if (fwrite(&item, sizeof(item), 1, fp) != 1)
    err_sys("fwrite error");
```

&emsp;&emsp;其中 *size* 为结构的大小，*nobj* 为元素的数量。

-----

&emsp;&emsp;fread 和 fwrite 返回读或写的对象数。对于读，如果出错或到达文件尾端，则此数字可以小于 *nobj*。在这种情况下，应该调用 ferror 或 feof 以判断究竟是哪种情况。对于写，如果返回值小于 *nobj*，则出错。

&emsp;&emsp;使用二进制 I/O 的基本问题是，它们只能用于读在同一系统上已写的数据：

1.   在一个结构中，同一成员的偏移量可能随着编译器和系统的不同而不同。
2.   用来存储多字节整数和浮点值的二进制格式在不同的系统结构间也可能不同。

## 定位流

&emsp;&emsp;有 3 种方法可以定位流：

1.   ftell 和 fseek 函数。它们假定文件位置可以存放在一个长整型中。
2.   ftello 和 fseeko 函数 (SUS 标准)。它们使用 off_t 数据类型替代长整型。
3.   fgetpos 和 fsetpos 函数 (ISO C)。它们使用抽象数据类型 fpos_t 记录文件位置。

&emsp;&emsp;可移植性最好的是第 3 对函数 fgetpos 和 fsetpos。

-----

```c
#include <stdio.h>

long ftell(FILE *fp);
/* Returns: current file position indicator if OK, −1L on error */

int fseek(FILE *fp, long offset, int whence);
/* Returns: 0 if OK, −1 on error */

void rewind(FILE *fp);
```

&emsp;&emsp;对于二进制文件，其文件位置指示器是从文件起始位置开始度量，并以字节为单位的。ftell 用于二进制文件时，其返回值就是这种字节位置。使用 fseek 定位一个二进制文件时，必须指定一个字节 *offset*，以及解释偏移量的方式 *whence*。*whence* 的值与 lseek 函数相同，但某些系统不支持 SEEK_END (UNIX 支持)。

&emsp;&emsp;对于文本文件，文件的当前位置可能不以简单的字节偏移量来度量。为了定位文本文件，*whence* 一定要是 SEEK_SET，而且 *offset* 一定只有两种值：0 (后退到文件起始位置)，或是对该文件的 ftell 所返回的值。函数 rewind 也可以将一个流设置到文件的起始位置。

&emsp;&emsp;UNIX 并不区分二进制文件和文本文件，所以，*whence* 的限制和 *offset* 的限制并不存在。

&emsp;&emsp;除了偏移量的类型是 off_t 而非 long 以外，函数 ftello 和 ftell 相同，函数 fseeko 与 fseek 相同：

```c
#include <stdio.h>

off_t ftello(FILE *fp);
/* Returns: current file position indicator if OK, (off_t)−1 on error */

int fseeko(FILE *fp, off_t offset, int whence);
/* Returns: 0 if OK, −1 on error */
```

&emsp;&emsp;ISO C 引入了 fgetpos 以及 fsetpos：

```c
#include <stdio.h>

int fgetpos(FILE *restrict fp, fpos_t *restrict pos);
int fsetpos(FILE *fp, const fpos_t *pos);

/* Both return: 0 if OK, nonzero on error */
```

&emsp;&emsp;fgetpos 将文件位置指示器的值存入 *pos* 指向的对象中。并且该值可作为以后调用 fsetpos 的参数。

## 格式化 I/O

1.   格式化输出

&emsp;&emsp;有 5 个 printf 函数来处理格式化输出：

```c
#include <stdio.h>

int printf(const char *restrict format, ...);
int fprintf(FILE *restrict fp, const char *restrict format, ...);
int dprintf(int fd, const char *restrict format, ...);

/* All three return: number of characters output if OK, negative value if output error */

int sprintf(char *restrict buf, const char *restrict format, ...);

/* Returns: number of characters stored in array if OK, negative value if encoding error */

int snprintf(char *restrict buf, size_t n, const char *restrict format, ...);

/* Returns: number of characters that would have been stored in array if buffer was large enough, negative value if encoding error */
```

&emsp;&emsp;printf 将格式化数据写出到标准输出，fprintf 写至指定的流，dprintf 写至指定的文件描述符，sprintf 将格式化字符送入数组 *buf* 中，sprintf 会在该数组的尾端自动添加一个 NULL 字符。

&emsp;&emsp;注意，sprintf 可能会造成缓冲区溢出。解决方案是，使用 snprintf 函数，在该函数中，缓冲区长度是一个显式参数，超出缓冲区长度的字符都将被丢弃。如果缓冲区足够大，snprintf 就会返回写入缓冲区的字符数。如果 snprintf 返回值是小于 *n* 的正数，那么就没有发生截断。

-----

&emsp;&emsp;格式说明符控制其余参数如何编写，以后该如何显示。转换说明符以 % 开始，除了转换说明符以外，格式字符串中的其他字符将按照原样被复制输出。一个转换说明符有 4 个可选择部分：

```c
%[flag][width][.precision][length]type
```

&emsp;&emsp;下面是 flag 的取值：

![](05.png)

&emsp;&emsp;width 说明最小字段宽度。若转换后的参数字符数小于宽度，则空余的字符位置用空格填充 (大于无妨)。width 应该是一个非负十进制整数，或者是一个星号。

&emsp;&emsp;precision 说明整型转换后最少输出数字位数，浮点数转换后小数点后的最少位数，字符串转换后最大字节数 (会截断)。精度是一个点 . 其跟一个可选的非负十进制数或一个星号。

&emsp;&emsp;宽度和精度都可以是星号 (*)。此时，一个整型参数指定宽度或精度的值，该整型参数正好位于被转换的参数之前。

&emsp;&emsp;length 说明参数长度，可能的值：

![](06.png)

&emsp;&emsp;type 是必填的，它控制如何解释参数：

![](07.png)

-----

&emsp;&emsp;下面 5 种 printf 函数变体类似上面 5 种，但是可变参数列表被替换为 *arg*：

```c
#include <stdarg.h>
#include <stdio.h>

int vprintf(const char *restrict format, va_list arg);
int vfprintf(FILE *restrict fp, const char *restrict format, va_list arg);
int vdprintf(int fd, const char *restrict format, va_list arg);
/* All three return: number of characters output if OK, negative value if output error */

int vsprintf(char *restrict buf, const char *restrict format, va_list arg);
/* Returns: number of characters stored in array if OK, negative value if encoding error */

int vsnprintf(char *restrict buf, size_t n, const char *restrict format, va_list arg);
/* Returns: number of characters that would have been stored in array if buffer was large enough, negative value if encoding error */
```

2.   格式化输入

&emsp;&emsp;有 3 个 scanf 函数执行格式化输入：

```c
#include <stdio.h>

int scanf(const char *restrict format, ...);
int fscanf(FILE *restrict fp, const char *restrict format, ...);
int sscanf(const char *restrict buf, const char *restrict format, ...);

/* All three return: number of input items assigned, EOF if input error or end of file before any conversion */
```

&emsp;&emsp;scanf 函数用于分析输入字符串，并将字符串序列转换成指定类型的变量。在格式化之后的参数包含了变量的地址，用转换结果对这些变量赋值。

&emsp;&emsp;格式说明符控制如何转换参数，以便对它们赋值。转换说明符以 % 开始，除转换说明符与空白字符以外，其他字符必须与输入字符串相匹配。若有一个字符不匹配，则停止后续处理，不再读其余部分：

```c
%[*][width][m][length]type
```

&emsp;&emsp;可选择的星号用于抑制转换，按照转换说明的其余部分进行转换，但是不将结果放入参数中。

&emsp;&emsp;除了 type 外，其余转换说明部分都类似 pritnf，type 的细微区别如下：

![](08.png)

-----

&emsp;&emsp;scanf 函数族也有可变长参数列表版本：

```c
#include <stdarg.h>
#include <stdio.h>

int vscanf(const char *restrict format, va_list arg);
int vfscanf(FILE *restrict fp, const char *restrict format, va_list arg);
int vsscanf(const char *restrict buf, const char *restrict format, va_list arg);

/* All three return: number of input items assigned, EOF if input error or end of file before any conversion */
```

## 实现细节

&emsp;&emsp;在 UNIX 中，标准 I/O 库最终都要调用系统 I/O 例程。每个标准 I/O 流都有一个与关联的文件描述符，可以使用 fileno 函数获得一个其描述符：

```c
#include <stdio.h>

int fileno(FILE *fp);

/* Returns: the file descriptor associated with the stream */
```

## 临时文件

&emsp;&emsp;ISO C 标准 I/O 库提供了两个函数来辅助创建临时文件：

```c
#include <stdio.h>

char *tmpnam(char *ptr);
/* Returns: pointer to unique pathname */

FILE *tmpfile(void);
/* Returns: file pointer if OK, NULL on error */
```

&emsp;&emsp;tmpnam 函数产生一个与现有文件名不同的一个有效路径名。每次调用它时，都产生一个不同的路径名，最多调用 TMP_MAX 次 (其定义在 <stdio.h> 中)。

&emsp;&emsp;如果 *ptr* 是 NULL，则所产生的路径名存放在一个静态区中，指向该静态区的指针作为返回值。后续调用 tmpnam 时会重写该静态区 (如果想调用多次，应该手工保存每次的字符串，而不是指针)。如果 *ptr* 不是 NULL，则其应该指向一个长度至少为 L_tmpnam 的字符数组，tmpnam 会把路径名存放在这个数组中并返回。

&emsp;&emsp;tmpfile 创建一个临时二进制文件 (类型为 wb+)，在关闭该文件或程序结束时将会被自动删除。

&emsp;&emsp;使用示例：

```c
#include "apue.h"
int main(void)
{
    char name[L_tmpnam], line[MAXLINE];
    FILE *fp;
    printf("%s\n", tmpnam(NULL)); /* first temp name */
    tmpnam(name);
    printf("%s\n", name);         /* second temp name */
    if ((fp = tmpfile()) == NULL)
        /* create temp file */
        err_sys("tmpfile error");
    fputs("one line of output\n", fp); /* write to temp file */
    rewind(fp);
    /* then read it back */
    if (fgets(line, sizeof(line), fp) == NULL)
        err_sys("fgets error");
    fputs(line, stdout);
    /* print the line we wrote */
}
```

&emsp;&emsp;执行结果：

```bash
$ ./a.out
/tmp/fileT0Hsu6
/tmp/filekmAsYQ
one line of output
```

&emsp;&emsp;tmpfile 容易想到的一个实现是：先调用 tmpnam 函数创建一个唯一的路径名，然后用该路径名创建一个文件，并立即 unlink 它 (注意，unlink 一个文件并不意味着立即删除它)。

&emsp;&emsp;SUS 的 XSI 扩展为处理临时文件定义了另外两个函数：

```c
#include <stdlib.h>

char *mkdtemp(char *template);
/* Returns: pointer to directory name if OK, NULL on error */

int mkstemp(char *template);
/* Returns: file descriptor if OK, −1 on error */
```

&emsp;&emsp;mkdtemp 函数创建一个临时且路径唯一的目录，成功则返回该目录的路径名。mkstemp 函数创建一个临时且路径唯一的文件，成功则返回该文件的描述符。二者目标的路径名都通过 *template* 字符串选择，该字符串的最后 6 位应该是 XXXXXX，函数将会修改这些占位符来构建一个唯一的路径名。

&emsp;&emsp;mkdtemp 函数创建的目录默认权限集为：S_IRUSR | S_IWUSR | S_IXUSR。调用进程可以通过修改 umask 来限制这些权限。mkstemp 函数创建一个临时文件并打开它，文件默认权限集为：S_IRUSR | S_IWUSR。

&emsp;&emsp;与 tmpfile 不同，mkstemp 函数创建的临时文件并不会自动删除，必须调用方手动解链。

&emsp;&emsp;应该使用 tmpfile 和 mkstemp 函数创建一个临时文件，而不是使用 tmpnam 和 open (creat) 的组合。理由是：后者不是一个原子操作。

&emsp;&emsp;使用例子：

```c
#include "apue.h"
#include <errno.h>

void make_temp(char *template);

int main()
{
    char char good_template[] = "/tmp/dirXXXXXX"; /* right way */
    *bad_template = "/tmp/dirXXXXXX";             /* wrong way*/
    printf("trying to create first temp file...\n");
    make_temp(good_template);
    printf("trying to create second temp file...\n");
    make_temp(bad_template);
}

void make_temp(char *template)
{
    int fd;
    struct stat sbuf;
    if ((fd = mkstemp(template)) < 0)
        err_sys("can’t create temp file");
    printf("temp name = %s\n", template);
    close(fd);
    if (stat(template, &sbuf) < 0) {
        if (errno == ENOENT)
            printf("file doesn’t exist\n");
        else
            err_sys("stat failed");
    } else {
        printf("file exists\n");
        unlink(template);
    }
}
```

&emsp;&emsp;运行结果：

```bash
$ ./a.out
trying to create first temp file...
temp name = /tmp/dirUmBT7h
file exists
trying to create second temp file...
Segmentation fault
```

&emsp;&emsp;第一个模板使用了字符数组，路径名在栈上分配。第二个模板使用了字符指针，只有指针自己留在栈上，而其所指向的字符串位于可执行文件的 .rodata 段上，当 mkstemp 试图修改该字符串时，将触发`段错误` (segment fault)。

## 内存流

&emsp;&emsp;SUS v4 中支持了内存流，其使用 FILE 指针的方式对内存进行进行访问，但其实并没有底层文件。

```c
#include <stdio.h>

FILE *fmemopen(void *restrict buf, size_t size, const char *restrict type);

/* Returns: stream pointer if OK, NULL on error */
```

&emsp;&emsp;fmemopen 函数允许调用者提供内存流的缓冲区：*buf* 指向缓冲区的开始位置，*size* 指定缓冲区的大小。如果 *buf* 为 NULL，fmemopen 将自动分配 *size* 大小的缓冲区，在这种情况下，流关闭时缓冲区会被自动释放。

&emsp;&emsp;*type* 参数控制如何使用流：

![](09.png)

&emsp;&emsp;这些取值类似打开文件时的标志，但是有微小差别：

1.   以追加写方式打开内存流时，当前文件位置会被设置为缓冲区中的第一个 NULL 字节。如果缓冲区没有 NULL 字节，则当前文件位置就被设置为缓冲区结尾的后一个字节。
2.   当流不以追加写方式打开内存流时，当前位置被设置为缓冲区开始的位置。

&emsp;&emsp;因此，内存流并不适合存储二进制数据，因为二进制数据中可能包含多个 NULL 字节。

3.   如果 *buf* 是 NULL，打开流进行读或写都没有任何意义。因为此时缓冲区是 fmemopen 函数自动分配的，没有办法获取缓冲区的地址：只写方式打开将无法读取已写入的数据，只读方式打开情况相反。
4.   当需要增加流缓冲区中数据量，以及调用 fclose，fflush，fseek，fseeko 和 fsetpos 时都会在当前文件位置写入一个 NULL 字节。

&emsp;&emsp;还有两个函数可以用于创建内存流：

```c
#include <stdio.h>
FILE *open_memstream(char **bufp, size_t *sizep);

#include <wchar.h>
FILE *open_wmemstream(wchar_t **bufp, size_t *sizep);

/* Both return: stream pointer if OK, NULL on error */
```

&emsp;&emsp;open_memstream 函数创建的内存流是字节定向的，open_wmemstream 函数创建的则是宽定向的，他们与 fmemopen 的区别在于：

1.   创建的流只能写打开。
2.   不能指定自己的缓冲区，但是可以通过 *bufp* 和 *sizep* 参数获得缓冲区地址和大小。
3.   关闭流后需要自行释放缓冲区。
4.   对流添加字节会增加缓冲区的大小。

&emsp;&emsp;使用他们必须遵守一些规则：缓冲区地址和大小只有在调用 fclose 和 fflush 后才有效。并且这些值只有在下一次流写入，或调用 fclose 前才有效。因为缓冲区可以增长，可能需要重新分配。

&emsp;&emsp;为了避免缓冲区溢出，内存流非常适合用于创建字符串。因为内存流只访问主存，不访问磁盘文件，所以对于把标准 I/O 作为参数用于临时文件的函数来说，会有很大的性能提升。
