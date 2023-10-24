---
title: APUE 07 - 进程环境
date: 2023-10-11 19:53:17
tags:
    - APUE
    - C
categories:
    - APUE
---

&emsp;&emsp;在了解进程控制原语前，需要先了解进程的环境。

<!-- more -->

## main 函数

&emsp;&emsp;C 程序总是从 main 函数开始执行，main 函数的原型是：

```c
int main(char argc, char *argv[]);
```

&emsp;&emsp;其中，*argc* 是命令行参数的数目，*argv* 是指向参数的各个指针所构成的数组。

&emsp;&emsp;当内核执行 C 程序时 (使用一个 exec 函数)，在调用 main 前要先调用一个特殊的启动例程 (函数)。可执行程序文件将此启动例程指定为程序的起始地址 - 这是链接器完成的。启动例程从内核取得命令行参数和环境变量值，为调用 main 函数做好准备。

## 进程终止

&emsp;&emsp;有 8 种方式使进程`终止` (termination)，其中 5 种是正常终止：

1.   从 main 返回。
2.   调用 exit。
3.   调用 _exit 或 _Exit。
4.   最后一个线程从启动例程返回。
5.   从最后一个线程调用 pthread_exit。

&emsp;&emsp;有 3 种异常的终止方式：

1.   调用 abort。
2.   接到一个信号。
3.   最后一个线程对取消请求作出响应。

&emsp;&emsp;启动例程通常是这样编写的：使得从 main 返回后立即调用 exit 函数。启动例程常常用汇编语言编写，但是可以用 C 代码形式表示为：

```c
exit(main(argc, argv));
```

1.   退出函数

&emsp;&emsp;有 3 个函数用于正常终止一个程序：_exit 和 _Exit 立即进入内核，exit 则先执行一些清理处理，然后进入内核：

```c
#include <stdlib.h>
void exit(int status);
void _Exit(int status);

#include <unistd.h>
void _exit(int status);
```

>   使用不同头文件的原因是：exit 和 _Exit 是 ISO C 说明的，而 _exit 是 POSIX 说明的。

&emsp;&emsp;由于历史原因，exit 函数总是执行一个标准 IO 库的清理关闭操作：对于所有打开流调用 fclose 函数，其作用是冲洗缓冲区上的所有数据。

&emsp;&emsp;3 个退出函数都带有一个整型参数，称为`终止状态` (或退出状态 exit status)。大多数 UNIX shell 都提供检查进程终止状态的方法。一般来说，如果没有显式指明终止状态，那么进程的退出状态是未定义的。但是 C99 规定，如果没有显式从 main 中 return，那么进程的终止状态是 0。

&emsp;&emsp;main 函数返回一个整型值与用该值调用 exit 是等价的：

```c
exit(0);

/* same as */

return 0;
```

2.   函数 atexit

&emsp;&emsp;ISO C 规定，一个进程可以登记多至 32 个函数，这些函数将由 exit 自动调用。这些函数被称为`终止处理程序` (exit handler)。可以调用 atexit 函数登记这些函数：

```c
#include <stdlib.h>

int atexit(void (*func)(void));

/* Returns: 0 if OK, nonzero on error */
```

&emsp;&emsp;其中，atexit 的参数是一个函数地址，当调用此函数时无需向它传递任何参数，也不期望它返回一个值。exit 调用这些函数的顺序与它们登记时候的顺序相反。同一函数如果被登记多次，也会被调用多次。

&emsp;&emsp;根据 ISO C 标准：exit 首先调用各个终止处理程序，然后关闭 (fclose) 所有打开流。POSIX 扩展了 ISO C 标准，它说明：如果程序调用 exec 函数，则将清理所有已登记的终止处理程序。下图描述一个 C 程序是如何启动的，以及它终止的各种方式：

![](01.png)

&emsp;&emsp;注意，内核使程序执行的唯一方式是调用一个 exec 函数。进程自愿终止的唯一方式是显式或隐式地 (通过 exit) 调用 _exit 或 _Exit。进程也可非自愿的由一个信号使其终止 (上图未显示)。

&emsp;&emsp;下面的程序说明如何使用 atexit 函数：

```c
#include "apue.h"

static void my_exit1(void);
static void my_exit2(void);

int main(void)
{
    if (atexit(my_exit2) != 0)
        err_sys("can’t register my_exit2");
    if (atexit(my_exit1) != 0)
        err_sys("can’t register my_exit1");
    if (atexit(my_exit1) != 0)
        err_sys("can’t register my_exit1");
    printf("main is done\n");
    return(0);
}

static void my_exit1(void)
{
    printf("first exit handler\n");
}

static void my_exit2(void)
{
    printf("second exit handler\n");
}
```

&emsp;&emsp;执行该程序输出：

```bash
$ ./a.out
main is done
first exit handler
first exit handler
second exit handler
```

## 命令行参数

&emsp;&emsp;执行一个程序时，调用 exec 的进程可以将命令行参数传递给新程序。

&emsp;&emsp;下面的程序将其所有的命令行参数回显到标准输出上：

```c
#include "apue.h"

int main(int argc, char *argv[])
{
    int
        i;
    for (i = 0; i < argc; i++)
        /* echo all command-line args */
        printf("argv[%d]: %s\n", i, argv[i]);
    exit(0);
}
```

&emsp;&emsp;程序运行结果：

```bash
$ ./echoarg arg1 TEST foo
argv[0]: ./echoarg
argv[1]: arg1
argv[2]: TEST
argv[3]: foo
```

&emsp;&emsp;ISO C 和 POSIX 都要求 argv[argc] 是一个空指针，因此可以改写循环为：

```c
for (i = 0; argv[i] != NULL; i++)
```

## 环境表

&emsp;&emsp;每个程序都接收到一张环境表，与命令行参数表一样，环境表也是一个字符指针数组，其中每个指针都指向一个以 NULL 字节结尾的 C 字符串。全局变量 environ 包含了该指针数组的地址：

```c
extern char **environ;
```

&emsp;&emsp;如果某个程序的环境有 5 个字符串，那么它看起来将如下图。变量 environ 被称为`环境指针` (environment pointer)，指针数组为环境表，其中的每个指针指向的字符串为`环境字符串`。

![](02.png)

&emsp;&emsp;按照惯例，环境由像下列形式的字符串组成：

```
name=value
```

&emsp;&emsp;在历史上，大多数 UNIX 系统支持 main 函数带 3 个参数，其中第 3 个参数就是环境表地址：

```c
int main(int argc, char *argv[], char *envp[]);
```

&emsp;&emsp;因为 ISO C 规定 main 函数只有两个参数，并且第 3 个参数与全局变量 environ 相比也没什么更大的好处。所以 POSIX 也规定使用全局变量 environ 而不是第 3 个参数。

&emsp;&emsp;通常，使用函数 getenv 和 putenv 来访问特定的环境变量，而不是直接使用 environ 变量。但是，如果要查看整个环境，则需使用 environ 指针。

## C 程序的存储空间布局

&emsp;&emsp;至今，C 程序一直由下列几部分组成：

-   正文段

&emsp;&emsp;正文段是 CPU 执行的机器指令的部分。通常，正文段是可共享的，所以即使是频繁执行的程序 (文本编辑器，C 编译器和 shell 等) 在存储器中也只需要有一个副本。并且，正文段往往是只读的，以防止程序由于意外而修改其指令。

-   初始化数据段

&emsp;&emsp;通常将此段称为数据段 (data 段)。它包含了程序中需明确赋予初始值的变量，例如在 C 程序中任何函数之外的声明 (全局变量)：

```c
int maxcount = 99;
```

-   未初始化数据段

&emsp;&emsp;通常将此段称为 bss 段，名称源自早期汇编程序的一个操作符，意思是`由符号开始的块` (block started by symbol)，在程序开始执行之前，内核将此段中的数据初始化为 0 或空指针。

```c
long sum[1000];
```

&emsp;&emsp;上面的全局变量声明，就使其被放在 bss 段中。

-   栈

&emsp;&emsp;自动变量以及每次函数调用时所需保存的信息都放在此段中。每次函数调用时，其返回地址以及调用者的环境信息 (如某些寄存器) 都存放在栈中。然后，最近被调用的函数在栈上为其自动和临时变量分配存储空间。通过以上方式使用栈，C 递归函数可以工作。

-   堆

&emsp;&emsp;堆常用于动态存储分配，由于历史原因，堆位于 bss 段与栈之间。

-----

&emsp;&emsp;这些段在程序中典型的逻辑布局方式：

![](03.png)

>   a.out 中还有若干其他类型的段，如包含符号表的段，包含调试信息的段以及包含动态共享库链接表的段等。这些部分并不装载到进程执行的程序映像中。

&emsp;&emsp;注意，bss 段的内容并不存放在磁盘程序文件中。这是因为，内核会在执行程序之前将他们设置为 0。需要存放在磁盘程序文件中的段只有正文段和 data 段。

&emsp;&emsp;size(1) 程序报告 text 段，data 段和 bss 段的长度：

```bash
$ size /usr/bin/cc /bin/sh
   text	   data	    bss	    dec	    hex	filename
1020441	  10320	  16152	1046913	  ff981	/usr/bin/cc
1364826	  48272	  45080	1458178	 164002	/bin/sh
```

## 共享库

&emsp;&emsp;共享库使得可执行文件中不需要再包含公共的库函数，而只需在所有进程都可引用的存储区中保存该库例程的一个副本。程序第一次执行或者第一次调用某个库函数时，用动态链接的方法将程序与共享库函数相链接。

&emsp;&emsp;这减少了每个可执行文件的长度，但增加了一些运行时间的开销。这种时间开销发生在该程序第一次被执行时，或者第一次调用该库函数时。共享库的另一个优点是可以用新版本的库函数替换旧版本，而无需对使用该库函数的程序重新编译 (假定库函数的参数数目和类型没有发生变化)。

&emsp;&emsp;使用静态库的程序大小：

```bash
$ gcc -static main.c # enforce gcc to use static libraries
$ ls -l a.out
-rwxr-xr-x. 1 arthur arthur 793744 Oct 13 14:05 a.out

$ size a.out        
   text    data     bss     dec     hex filename
 628709   20880   22728  672317   a423d a.out
```

&emsp;&emsp;对比使用动态链接的可执行文件：

```bash
$ gcc main.c # gcc defaults to use shared libraries
$ ls -l a.out
-rwxr-xr-x. 1 arthur arthur 16680 Oct 13 14:06 a.out

$ size a.out
   text    data     bss     dec     hex filename
   1027     532       4    1563     61b a.out
```

## 存储空间分配

&emsp;&emsp;ISO C 定义了 3 个用于存储空间动态分配的函数：

1.   malloc：分配指定字节数的存储区，此存储区中的初始值不确定。
2.   calloc：为指定数量指定长度的对象分配存储空间，该空间中的每一位都初始化为 0。

3.   realloc：增加或减少以前分配区的长度。当增加长度时，可能需要将以前分配区的内容转移到另一个足够大的区域，以便在尾端提供增加的存储区，而新增区域内的初始值不确定。

```c
#include <stdlib.h>

void *malloc(size_t size);
void *calloc(size_t nobj, size_t size);
void *realloc(void *ptr, size_t newsize);

/* All three return: non-null pointer if OK, NULL on error */

void free(void *ptr);
```

&emsp;&emsp;这 3 个分配函数返回的指针一定是适当对齐的，使其可用于任何数据对象。

&emsp;&emsp;这 3 个函数的返回值类型都是 void *，如果使用前导入了 <stdlib.h> 获得其原型，那么可以将其返回的指针隐式转换成任意指针类型。如果没有导入其函数签名，那么默认函数的返回值类型为 int。此时，从 int 转换到指针类型就可能引发错误。

&emsp;&emsp;free 函数释放 *ptr* 指向的存储空间。被释放的空间通常被送入可用存储区池。以后，可在调用上面 3 个分配函数时再分配。

&emsp;&emsp;realloc 函数可以增减以前分配的存储区长度。在增加存储区长度时，如果原存储区域后面有足够的空间执行原地扩充，那么不会改变存储区的起始地址。如果存储区后面的空间不足以扩充，realloc 则必须寻找一片新的区域来包含原存储区域和新增的区域，并且 realloc 必须将原区域的数据复制到新区域，此时，返回的存储区起始地址将不同于 *ptr*。特殊的，如果 *ptr* 是 NULL，此时 realloc 行为与 malloc 一致。

-----

&emsp;&emsp;这些分配函数通常使用 sbrk(2) 系统调用实现，该系统调用扩充 (或缩小) 进程的堆。虽然 sbrk 可以扩充和缩小进程的堆空间，但是大多数 malloc 和 free 的实现都不缩小堆，因为释放的空间可供以后再分配，所以将他们保存在 malloc 池中而不是返回给内核。

&emsp;&emsp;大多数实现所分配的存储空间比所要求的稍大一些，额外的空间用来记录管理信息和对齐 (分配块的长度，指向下一个分配块的指针等)。这带来一个问题：如果超过一个已分配区的尾端，或在已分配区的起始位置之前进行写操作，则会改写另一块的管理记录信息。

&emsp;&emsp;其他可能的致命错误是：释放一个已经释放的块。传给 free 的指针不是 3 个分配函数的返回值。如果一个进程调用 malloc 函数，却忘记调用 free 函数。则该进程占用的存储空间就会连续增加，这称为`泄漏` (leakage)。严重的内存泄漏将导致进程的地址空间长度不断增长，直至没有空闲空间。并且，由于过度的换页开销，会造成性能下降。

## 替代的存储空间分配程序

&emsp;&emsp;有很多可替代 malloc 和 free 的函数：

1.   函数 alloca

&emsp;&emsp;alloca 函数的调用序列与 malloc 相同，但是它在当前函数的栈帧上分配存储空间，而不是在堆中。其优点是：当函数返回时，自动释放它所使用的栈帧，所以不必为释放空间而费心。其缺点是：alloca 函数增加了函数栈帧的长度。

## 环境变量

&emsp;&emsp;环境字符串的形式是：

```
name=value
```

&emsp;&emsp;UNIX 内核并不查看这些字符串，它们的解释完全取决于各个应用程序。典型的例子是 shell，它使用了大量的环境变量。其中的一些在登录时自动设置 (HOME USER 等)，有些则是由用户设置。

&emsp;&emsp;ISO C 定义了一个函数 getenv，可以用来获取环境变量值：

```c
#include <stdlib.h>

char *getenv(const char *name);

/* Returns: pointer to value associated with name, NULL if not found */
```

&emsp;&emsp;getenv 返回一个指针，它指向 *name=value* 字符串中的 *value* 部分。

&emsp;&emsp;SUS 中的 POSIX 定义了某些环境变量：

![](04.png)

&emsp;&emsp;除了获取环境变量外，也能设置环境变量：

![](05.png)

```c
#include <stdlib.h>

int putenv(char *str);
/* Returns: 0 if OK, nonzero on error */

int setenv(const char *name, const char *value, int rewrite);
int unsetenv(const char *name);

/* Both return: 0 if OK, −1 on error */
```

&emsp;&emsp;这 3 个函数的操作是：

1.   putenv 取形式为 *name=value* 的字符串，并将其放到环境变量表中。如果环境中 *name* 已经存在，则删除原来的定义。
2.   setenv 将 *name* 设置为 *value*。如果环境中已经存在 *name*：若 *rewrite* 非 0，则先删除原来的定义。若 *rewrite* 为 0，则既不删除原来的定义也不更新 *value*。
3.   unsetenv 删除 *name* 的定义。即使不存在 *name* 也不算出错。

>   注意 putenv 和 setenv：setenv 必须分配存储空间，以便依据其参数创建 *name=value* 字符串。putenv 可以简单地将传递给他的指针参数直接放到环境表中。因此，将栈中的字符串作为参数传递给 putenv 可能会引发错误。

&emsp;&emsp;如果是删除一个已存在的 *name*，那么只需在环境表中找到其关联的环境字符串指针，然后将其后面的指针都向前移动一位即可。

&emsp;&emsp;但是修改或创建一个新的环境字符串就麻烦许多：因为环境表和环境字符串通常占据的是进程地址空间的顶部，所以它不能再向高处或低处扩展。

1.   如果修改一个已有的 *name*：
     -   如果新 *value* 的长度小于等于现有 *value*，则只需将新字符串复制到原字符串占据的空间中。
     -   如果新 *value* 的长度大于现有 *value*，则必须调用 malloc 为新字符串 (包括 *name*) 分配空间，然后将该字符串复制到新空间中，接着使环境表中原 *value* 的指针指向新空间。
2.   如果要增加一个新 *name*，则必须先调用 malloc 为新字符串 (包括 *name*) 分配空间并将该字符串复制到新空间中，然后：
     -   如果是第一次增加 *name*，则必须再调用 malloc 为新环境表分配空间。然后，将旧的环境表复制到新环境表中，并将指向 *name=value* 的字符串指针插入新环境表表尾 (最末尾是 NULL)。最后，修改全局变量 environ，使其指向新环境表的第一个表项。
     -   如果不是第一次增加 *name*，则可知之前已经调用过 malloc 分配新环境表了，因此只需调用 realloc，将环境表空间扩大一个指针的空间，然后将指向 *name=value* 的字符串指针插入表尾 (最末尾是 NULL)。如果 realloc 结果不与原环境表地址相同，则还需更新 environ 变量。

## 函数 setjmp 和 longjmp

&emsp;&emsp;C 语言中，局部跳转 goto 不能跨越函数，但是函数 setjmp 和 longjmp 却可以跨函数跳转。这两个函数在跳出深层函数嵌套时十分有用。

&emsp;&emsp;一个简单的例子：

```c
#include "apue.h"
#define TOK_ADD

void do_line(char *);
void cmd_add(void);
int get_token();

int main(void)
{
    char line[MAXLINE];
    while (fgets(line, MAXLINE, stdin) != NULL)
        do_line(line);
    exit(0);
}

char *tok_ptr;

void do_line(char *ptr)
{
    int cmd;
    /* global pointer for get_token() */
    /* process one line of input */
    tok_ptr = ptr;
    while ((cmd = get_token()) > 0) {
        switch (cmd) { /* one case for each command */
        case TOK_ADD:
            cmd_add();
            break;
        }
    }
}

void cmd_add(void)
{
    int token;
    token = get_token();
    /* rest of processing for this command */
}

int get_token(void)
{
    /* fetch next token from line pointed to by tok_ptr */
}
```

&emsp;&emsp;自动变量存储在每个函数的栈帧上，上面程序的函数调用栈：

![](06.png)

&emsp;&emsp;一个棘手的问题是，当程序遇到非致命错误时：比如 cmd_add 函数检测到输入的数无效，它该如何返回 main 函数。因为 cmd_add 在调用栈上位于 do_line 之后，它无法直接返回 main，所以它必须传递一个有特殊含义的返回值给 do_line，do_line 接收到这个返回值后再转发给 main 函数。

&emsp;&emsp;上面的方法会使程序逻辑变得冗长，另一个解决办法是：使用非局部跳转 setjmp 和 longjmp 函数。它允许直接在栈上跳过若干调用帧，返回到当前函数调用路径上的某一个函数中。

```c
#include <setjmp.h>

int setjmp(jmp_buf env);
/* Returns: 0 if called directly, nonzero if returning from a call to longjmp */

void longjmp(jmp_buf env, int val);
```

&emsp;&emsp;在希望返回到的位置调用 setjmp (本例为 main 函数)。直接调用 setjmp 时，其返回值是 0。setjmp 的参数 *env* 是一个特殊类型 jmp_buf，它是一个某种类型的数组，其中存放了在调用 longjmp 后能用来恢复栈状态的所有信息。因为要在不同函数中使用它，所以通常将 *env* 定义为全局变量。

&emsp;&emsp;当检查到一个错误时，以两个参数调用 longjmp 函数：第一个是调用 setjmp 时所用到的 *env*，第二个是一个非 0 值 *val*，它将成为从 setjmp 处返回的值。使用 *val* 的原因是：一个 setjmp 可以有多个 longjmp，当返回到 setjmp 时，*val* 可以用来区分是从哪个 longjmp 返回的。

&emsp;&emsp;下面是修改过后的例子：

```c
#include "apue.h"
#include <setjmp.h>
#define TOK_ADD 5

jmp_buf jmpbuffer; 

int main(void)
{
    char line[MAXLINE];
    if (setjmp(jmpbuffer) != 0)
        printf("error");
    while (fgets(line, MAXLINE, stdin) != NULL)
        do_line(line);
    exit(0);
}
...
void cmd_add(void)
{
    int token;
    token = get_token();
    if (token < 0)
        /* an error has occurred */
        longjmp(jmpbuffer, 1);
    /* rest of processing for this command */
}
```

&emsp;&emsp;函数 main 调用 setjmp 时，它将所需的信息记入变量 jmpbuffer 中并返回 0。然后调用 do_line，后者又调用 cmd_add。此时函数调用栈如图：

![](07.png)

&emsp;&emsp;如果 cmd_add 检测到一个错误，调用了 longjmp。此时 longjmp 将使栈反绕到 main 的栈帧，也就是抛弃了 cmd_add 和 do_line 的栈帧。调用 longjmp 造成 main 中的 setjmp 返回，但是，这一次 setjmp 的返回值是 1 (longjmp 的参数 *val*)。

1.   自动变量，寄存器变量和易失变量

&emsp;&emsp;下面的程序说明了在调用 longjmp 之后，自动变量，寄存器变量，静态变量和易失变量的不同情况：

```c
#include <stdio.h>
#include <setjmp.h>
#include <stdlib.h>

static void f1(int, int, int, int);
static void f2(void);

static jmp_buf jmpbuffer;
static int globval;

int main(void)
{
    int autoval;
    register int regival;
    volatile int volaval;
    static int statval;

    globval = 1;
    autoval = 2;
    regival = 3;
    volaval = 4;
    statval = 5;

    if (setjmp(jmpbuffer) != 0) {
        printf("after longjmp:\n");
        printf("globval = %d, autoval = %d, regival = %d,"
               " volaval = %d, statval = %d\n",
               globval, autoval, regival, volaval, statval);
        exit(0);
    }

    /* Change variables after setjmp, but before longjmp. */
    globval = 95;
    autoval = 96;
    regival = 97;
    volaval = 98;
    statval = 99;
    f1(autoval, regival, volaval, statval); /* never returns */
    exit(0);
}

static void f1(int i, int j, int k, int l)
{
    printf("in f1():\n");
    printf("globval = %d, autoval = %d, regival = %d,"
           " volaval = %d, statval = %d\n",
           globval, i, j, k, l);
    f2();
}

static void f2(void)
{
    longjmp(jmpbuffer, 1);
}
```

&emsp;&emsp;分别以不带优化和带优化的选项编译运行：

```bash
$ gcc -O0 main.c && ./a.out
in f1():
globval = 95, autoval = 96, regival = 97, volaval = 98, statval = 99
after longjmp:
globval = 95, autoval = 96, regival = 3, volaval = 98, statval = 99

$ gcc -O  main.c && ./a.out
in f1():
globval = 95, autoval = 96, regival = 97, volaval = 98, statval = 99
after longjmp:
globval = 95, autoval = 2, regival = 3, volaval = 98, statval = 99
```

&emsp;&emsp;注意：全局变量，静态变量和易失变量不受优化的影响，它们的值总是在调用 longjmp 时最近所呈现的值。但是，自动变量和寄存器变量的行为有所区别，寄存器变量一般会被保存在寄存中，所以在调用 longjmp 时可能会被恢复。而自动变量可优化的空间比较大，如果允许，编译器也会把自动变量放在寄存器中，此时，调用 longjmp 也会恢复自动变量的值。

2.   自动变量潜在的问题

&emsp;&emsp;基于栈的自动变量会随着栈的销毁而被一起销毁，所以：返回栈上自动变量的指针是不安全的。

```c
#include "apue.h"

FILE *open_data(void)
{
    FILE *fp;
    char databuf[BUFSIZ];
    /* setvbuf makes this the stdio buffer */
    if ((fp = fopen("datafile", "r")) == NULL)
        return(NULL);
    if (setvbuf(fp, databuf, _IOLBF, BUFSIZ) != 0)
        return(NULL);
    return(fp);
    /* error */
}
```

&emsp;&emsp;当 open_data 返回时，它之前占据的栈空间将被下一个调用函数所使用。此时，databuf 指向的内容就是不确定的。修改方案是：应在全局存储空间 (static 或 extern) 或者动态的 (alloc) 为 databuf 分配空间。

## 函数 getrlimit 和 setrlimit

&emsp;&emsp;每个进程都有一组资源限制，其中一些可以用 getrlimit 和 setrlimit 函数查询和更改。

```c
#include <sys/resource.h>

int getrlimit(int resource, struct rlimit *rlptr);
int setrlimit(int resource, const struct rlimit *rlptr);

/* Both return: 0 if OK, −1 on error */
```

>   这两个函数由 SUS 的 XSI 扩展定义。进程的资源限制通常是在系统初始化时由 0 进程建立的，然后由后继进程继承。

&emsp;&emsp;这两个函数每次调用都要指定一个资源以及一个指向下列结构的指针：

```c
struct rlimit {
    rlim_t rlim_cur;  /* soft limit: current limit */
    rlim_t rlim_max;  /* hard limit: maximum value for rlim_cur */
};
```

&emsp;&emsp;更改进程资源时，要遵守下列 3 条规则：

1.   任何进程都可以将其软限制更改为一个小于等于硬限制的值。
2.   任何进程都可以降低其硬限制值，但它必须大于等于其软限制值。并且，对于普通用户来说，降低硬限制是不可逆的。
3.   只有超级用户可以提高硬限制值。

&emsp;&emsp;常量 RLIM_INFINITY 指定了一个无限量的限制值。

&emsp;&emsp;这两个函数的 *resource* 参数取值如下：

![](08.png)

| resource          | meaning                                                      |
| ----------------- | ------------------------------------------------------------ |
| RLIMIT_AS         | 进程总的可用内存最大长度，这将影响 sbrk 和 mmmap 函数。      |
| RLIMIT_CORE       | core 文件的最大字节数，为 0 时将阻止创建 core 文件。         |
| RLIMIT_CPU        | CPU 时间的最大量值 (秒)，当超过此软限制时，向该进程发送 SIGXCPU 信号。 |
| RLIMIT_DATA       | 数据段的最大字节长度，包括 data 段和 bss 段。                |
| RLIMIT_FSIZE      | 可以创建的文件的最大字节长度，当超过此软限制时，向该进程发送 SIGXFSZ 信号。 |
| RLIMIT_MEMLOCK    | 一个进程使用 mlock(2) 能够锁定在内存中的最大字节长度。       |
| RLIMIT_MSGQUEUE   | 进程为 POSIX 消息队列可分配的最大内存字节数。                |
| RLIMIT_NICE       | 为了影响进程的调度优先级，nice 值可设置的最大限制。          |
| RLIMIT_NOFILE     | 每个进程能打开的最多文件数，更改此限制将影响 sysconf 函数在参数 _SC_OPEN_MAX 中的返回值。 |
| RLIMIT_NPROC      | 每个实际用户 ID 可用有的最大子进程数。更改此限制将影响 sysconf 函数在参数 _SC_CHILD_MAX 中的返回值。 |
| RLIMIT_NPTS       | 用户可同时打开的伪终端的最大数量。                           |
| RLIMIT_RSS        | 最大驻内存集字节长度，如果可用的物理存储器非常少，则内核将从进程处取回超过 RSS 的部分。 |
| RLIMIT_SBSIZE     | 在任一给定时刻，一个用户可以占用的套接字缓冲区的最大字节长度。 |
| RLIMIT_SIGPENDING | 一个进程可排队的信号最大数量，这个限制是 sigqueue 函数实施的。 |
| RLIMIT_STACK      | 栈的最大字节长度。                                           |
| RLIMIT_SWAP       | 用户可消耗的交换空间的最大字节数。                           |
| RLIMIT_VMEM       | 这是 RLIMIT_AS 的同义词。                                    |

