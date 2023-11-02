---
title: CSAPP 12 - 并发编程
date: 2023-09-06 17:25:27
updated: 2023-09-15 19:00:00
tags:
  - CSAPP
  - C
categories:
  - CSAPP
---

&emsp;&emsp;如果逻辑控制流在时间上重叠，那么它们就是并发的。并发出现在计算机系统许多层面上，使用应用级并发的程序称为并发程序。

<!-- more -->

## 基于进程

&emsp;&emsp;构造并发程序最简单的方法就是用进程。一个构造并发服务器的自然方法是：在父进程中接受客户端的链接请求，然后创建一个新的子进程来为每个新客户端提供服务。

&emsp;&emsp;假设有两个客户端和一个服务器，服务器在监听一个监听描述符 (3) 上的连接请求。现在假设服务器接受了客户端 1 的连接请求，并返回一个已连接描述符 (4)。在接受连接请求后，服务器派生一个子进程，这个子进程获得服务器进程描述符表的完整副本。子进程关闭它的副本中的描述符 3，而父进程关闭它的已连接描述符 4 的副本。

![](01.png)

&emsp;&emsp;父子进程的已连接描述符都指向一个文件表表项，所以父进程必须关闭它的已连接描述符的副本。否则，将永不会释放描述符 4 的文件表条目，这会引起内存泄漏。

&emsp;&emsp;现在，假设父进程为客户端 1 创建了子进程后，它接受一个新的客户端 2 的连接请求，并返回一个新的已连接文件描述符 (5)，然后父进程又派生一个新的子进程，这个子进程用于为客户端 2 提供服务：

![](02.png)

### 并发服务器

&emsp;&emsp;将前一章中的 echo 服务器修改为基于进程的并发服务器，有以下注意：

- 通常服务器都会运行很长时间，所以必须包括一个 SIGCHLD 处理程序，来回收僵死子进程的资源。因为 Linux 信号是不排队的，所以 SIGCHLD 处理程序必须准备好回收多个僵死子进程的资源。
- 父子进程必须关闭它们各自的 connfd，以避免内存泄漏。
- 因为套接字文件表表项中的引用技术，只有父子进程的 connfd 都关闭了，到客户端的连接才会终止。

```c
#include "assist.h"

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>
#include <sys/socket.h>

void sigchld_handler(int sig)
{
    while (waitpid(-1, 0, WNOHANG) > 0)
        ;
}

int main(int argc, char **argv)
{
    int listen_fd, client_fd;
    socklen_t client_len;

    struct sockaddr_storage client_addr;
    char client_hostname[MAXLINE], client_port[MAXLINE];

    if (argc != 2) {
        fprintf(stderr, "usage: %s <port>\n", argv[0]);
        exit(0);
    }

    signal(SIGCHLD, sigchld_handler);
    listen_fd = open_listen_fd(argv[1]);
    if (listen_fd == -1) {
        fprintf(stderr, "cannot create a listen socket\n");
        exit(1);
    }

    for (;;) {
        client_len = sizeof(client_addr);
        client_fd = accept(listen_fd, (SA)&client_addr, &client_len);

        if (fork() == 0) {
            close(listen_fd);
            echo(client_fd);
            close(client_fd);
            exit(0);
        }

        fprintf(stderr, "new connection\n");
        close(client_fd);
    }
}
```

### 进程的优劣

&emsp;&emsp;对于在父子进程间共享状态，进程有一个非常清晰的模型。共享文件表，但是不共享用户地址空间。进程有独立的地址空间既是优点也是缺点。这样一来，一个进程不可能不小心覆盖另一个进程的虚拟内存，这是一个优点。

&emsp;&emsp;另一方面，独立的地址空间会使得进程共享状态信息变得更加困难，为了共享信息，它们必须使用显示的 IPC 机制。基于进程的设计的另一个缺点是，它们往往比较慢，因为进程控制和 IPC 的开销很高。

## I/O 多路复用

&emsp;&emsp;如果 echo 服务器也要能对用户从标准输入键入的命令作出响应，那么服务器必须响应两个独立的 I/O 事件，问题是无法确定先等待哪个事件。一个解决方案是使用 I/O 多路复用技术。基本思想就是使用 select 函数，要求内核挂起进程，只有在一个或多个 I/O 事件发生后，才将控制返回给进程。

&emsp;&emsp;select 函数比较复杂，有许多不同的使用场景，这里只讨论第一种场景：等待一组描述符准备好读。

```c
int select(int n, fd_set *fdset, NULL, NULL, NULL);

FD_ZERO(fd_set *fdset);
FD_CLR(int fd, fd_set *fdset);
FD_SET(int fd, fd_set *fdset);
FD_ISSET(int fd, fd_set *fdset);
```

&emsp;&emsp;select 函数处理类型为 fd_set 的集合，也叫做描述符集合，逻辑上，可以将描述符集合看成一个大小为 n 的位向量：b~n-1~ ... b~1~ b~0~。

&emsp;&emsp;其中，每个位 b~k~ 对应描述符 k。当且仅当 b~k~ = 1，描述符 k 才表明是集合的一个元素。对 fd_set 只能有三种操作：分配，复制，使用上述宏。

&emsp;&emsp;select 函数的参数中，第二个参数就是读描述符集合。第一个参数是描述符集合的基数 (任何描述符集合的最大基数) 其实就是要监听的最大描述符 n + 1。调用 select 函数会一直阻塞，直到读集合中至少有一个描述符准备好可以读。select 函数会修改参数 fdset，修改后 fdset 就是读集合的一个子集，称为准备好集合。

&emsp;&emsp;一个简单的例子：

```c
#include "assist.h"

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/socket.h>

void command();

int main(int argc, char **argv)
{
    int listen_fd, client_fd;
    socklen_t client_len;

    struct sockaddr_storage client_addr;
    char client_hostname[MAXLINE], client_port[MAXLINE];

    fd_set read_set, ready_set;

    if (argc != 2) {
        fprintf(stderr, "usage: %s <port>\n", argv[0]);
        exit(0);
    }

    listen_fd = open_listen_fd(argv[1]);
    if (listen_fd == -1) {
        fprintf(stderr, "cannot create a listen socket\n");
        exit(1);
    }

    FD_ZERO(&read_set);
    FD_SET(STDIN_FILENO, &read_set);
    FD_SET(listen_fd, &read_set);

    for (;;) {
        ready_set = read_set;
        select(listen_fd + 1, &ready_set, NULL, NULL, NULL);

        if (FD_ISSET(STDIN_FILENO, &ready_set))
            command();

        if (FD_ISSET(listen_fd, &ready_set)) {
            fprintf(stderr, "new connection\n");
            client_len = sizeof(client_addr);
            client_fd = accept(listen_fd, (SA *)&client_addr, &client_len);
            echo(client_fd);
        }

        close(client_fd);
    }
}

void command()
{
    char buf[MAXLINE];
    if (!fgets(buf, MAXLINE, stdin))
        exit(0);
    fprintf(stderr, "from console: %s", buf);
}
```

&emsp;&emsp;现在这个服务器可以同时等待两个 I/O 事件了，但是它还有很多缺点。

### 并发服务器

&emsp;&emsp; I/O 多路复用可以用作**并发事件驱动服务器**程序的基础，在事件驱动程序中，某些事件会导致流向前推进。一般的思想是将逻辑流模型化为状态机。`状态机`一般指有限状态机 FSM 是一种数学模型，它包含四部分要素：状态，事件，动作，转移。

&emsp;&emsp;对于每个新的客户端 k，服务器会创建一个新的状态机 s~k~，并将它和已连接描述符 d~k~ 联系起来。在任何时刻，状态机 s~k~ 都处于某个确定的状态下，当事件发生时，可能会有一些动作被执行，并且状态机的状态可能被转移到另一个状态。

&emsp;&emsp;具体来说，服务器借助 select 函数来检测事件的发生，当每个已连接描述符准备好可读时，服务器就为相应的状态机执行转移：

```c
typedef struct {
    int maxi;                 /* high water index into client array */
    int maxfd;                /* largest fd in read_set */
    int nready;               /* number of ready fd from select */
    fd_set read_set;          /* set of all active fds */
    fd_set ready_set;         /* subset of fds ready for reading */
    int clientfd[FD_SETSIZE]; /* set of active fds */
} pool;

extern void init_pool(int listen_fd, pool *pool);

extern void add_client(int fd, pool *pool);

extern void check_clients(pool *pool);
```

&emsp;&emsp;在本例中，服务器首先创建一个 pool 结构并调用 init_pool 进行初始化。随后服务器进入无限循环，检测读集合中的事件：如果是监听套接字上发生事件，那么就调用 add_client 函数处理。然后调用 check_clients 函数，该函数主要检测是否有客户端消息，有则处理。

```c
void init_pool(int listen_fd, pool *pool)
{
    pool->maxi = -1;
    for (int i = 0; i < FD_SETSIZE; i++)
        pool->clientfd[i] = -1;

    pool->maxfd = listen_fd;
    FD_ZERO(&pool->read_set);
    FD_SET(listen_fd, &pool->read_set);
}

void add_client(int fd, pool *pool)
{
    pool->nready--;
    for (int i = 0; i < FD_SETSIZE; i++)
        if (pool->clientfd[i] < 0) {
            /* add client fd to the pool */
            pool->clientfd[i] = fd;
            FD_SET(fd, &pool->read_set);

            if (fd > pool->maxfd)
                pool->maxfd = fd;

            if (i > pool->maxi)
                pool->maxi = i;

            break;
        }
}

void check_clients(pool *pool)
{
    int client_fd, n;
    char buf[MAXLINE];

    for (int i = 0; (i <= pool->maxi) && (pool->nready > 0); i++) {
        client_fd = pool->clientfd[i];

        if (client_fd > 0 && FD_ISSET(client_fd, &pool->ready_set)) {
            pool->nready--;

            if ((n = read(client_fd, buf, MAXLINE - 1)) > 0) {
                /* write back to client */
                write(client_fd, buf, n);
                /* print msg into screen */
                buf[n] = 0;
                fprintf(stderr, "from client: %s", buf);

            } else { /* EOF or Errors */
                close(client_fd);
                FD_CLR(client_fd, &pool->read_set);
                pool->clientfd[i] = -1;
            }
        }
    }
}
```

&emsp;&emsp;clientfd 数组记录了已连接的描述符集合 (-1 代表空)，maxi 指明了其中的最大索引。

&emsp;&emsp;服务器代码：

```c
#include "assist.h"

#include <stdio.h>
#include <stdlib.h>
#include <sys/socket.h>

int main(int argc, char **argv)
{
    int listen_fd, client_fd;
    socklen_t client_len;
    static pool pool;

    struct sockaddr_storage client_addr;
    char client_hostname[MAXLINE], client_port[MAXLINE];

    if (argc != 2) {
        fprintf(stderr, "usage: %s <port>\n", argv[0]);
        exit(0);
    }

    listen_fd = open_listen_fd(argv[1]);
    if (listen_fd == -1) {
        fprintf(stderr, "cannot create a listen socket\n");
        exit(1);
    }

    init_pool(listen_fd, &pool);
    for (;;) {
        pool.ready_set = pool.read_set;
        pool.nready = select(pool.maxfd + 1, &pool.ready_set, NULL, NULL, NULL);

        if (FD_ISSET(listen_fd, &pool.ready_set)) {
            client_len = sizeof(client_addr);
            client_fd = accept(listen_fd, (SA *)&client_addr, &client_len);
            fprintf(stderr, "new connection\n");
            add_client(client_fd, &pool);
        }

        check_clients(&pool);
    }
}
```

### 比较和优劣

&emsp;&emsp;事件驱动设计的一个优点是：它比基于进程的设计给程序员更多的对程序行为的控制。另一个优点是，一个基于 I/O 多路复用的事件驱动服务器是运行在单一进程上下文中的，因此每个逻辑流都能访问该进程的全部地址空间。这使得在流之间共享数据变得很容易。此外，单进程程序还更便于调试。

&emsp;&emsp;事件驱动设计的一个明显缺点就是编码复杂，并且随着粒度的减小，复杂性还会上升。这里的粒度指：每个逻辑流每个时间片执行的指令数量。

## 基于线程

&emsp;&emsp;线程就是运行在进程上下文中的逻辑流。现代系统允许一个进程里同时运行多个线程，线程是由内核自动调度的。每个线程都有他自己的线程上下文，包括一个唯一的整数线程 ID，栈，栈指针，程序计数器，通用寄存器和条件码。所有运行在一个进程里的线程共享该进程的整个虚拟地址空间。

&emsp;&emsp;基于线程的逻辑流结合了基于进程和基于 I/O 多路复用的流的特性。同进程一样，线程由内核自动调度，并且内核通过线程 ID 来识别线程。同基于 I/O 多路复用技术的流一样，多个线程运行在单一进程的上下文中，因此共享这个进程虚拟地址空间的所有内容，包括代码，数据，共享库，打开文件等。

### 线程执行模型

&emsp;&emsp;多线程的执行模型在某些方面和多进程的相似。每个进程开始生命周期时都是单一线程，这个线程叫做`主线程`。在某一时刻，主线程创建一个`对等线程`，从这个时间点开始，两个线程就并发的开始运行。

![](03.png)

&emsp;&emsp;在一些重要的方面，线程的执行是不同于进程的。线程的上下文比进程的上下文小很多，因此线程的上下文切换要比进程快得多。此外，线程不像进程一样按照严格的父子层次来组织。和一个进程相关的线程组成一个对等线程池 (对等池)，独立于其他线程创建的线程。主线程和其他线程的区别仅在于它总是进程中第一个运行的线程。

### Posix 线程

&emsp;&emsp;Posix 线程是在 C 程序中处理线程的一个标准接口。Pthreads 定义了大约 60 个函数，允许程序创建，杀死和回收线程，与对等线程安全的共享数据，还可以通知对等线程系统状态的变化。

#### 创建线程

&emsp;&emsp;线程通过调用 pthread_create 函数来创建其他线程。

```c
int pthread_create(pthread_t *tid, pthread_attr_t *attr, func* f, void *arg);
```

&emsp;&emsp;pthread_create 函数创建一个新的线程，并带着一个输入变量 arg，在新线程上运行**线程例程 f**，能用 attr 参数来改变创建线程的默认属性。

&emsp;&emsp;pthread_create 函数返回时，参数 tid 包含新创建线程的 ID。新线程可以通过 pthread_self 函数来获得自己的线程 ID：

```c
pthread_t pthread_self(void);
```

#### 终止线程

&emsp;&emsp;一个线程以下列方式之一来终止：

- 顶层的线程例程返回时，线程会**隐式终止**。
- 通过调用 pthread_exit 函数，线程会显示的终止。如果主线程调用 pthread_exit，它会等待所有其他对等线程终止，然后再终止主线程和整个进程。返回值为 pthread_return。

```c
void pthread_return(void *thread_return);
```

- 如果某个对等线程调用 exit 函数，该函数会终止进程以及所有与进程相关的线程。

- 另一个对等线程通过以当前线程 ID 作为参数调用 pthread_cancel 函数来终止当前线程。

```c
int pthread_cancel(pthread_t tid);
```

#### 回收线程资源

&emsp;&emsp;线程通过调用 pthread_join 函数等待其他线程终止。

```c
int pthread_join(pthread_t tid, void **thread_return);
```

&emsp;&emsp;pthread_join 函数会阻塞，直到线程 tid 终止，将线程例程返回的通用指针赋值到 thread_return 指向的位置，然后回收已终止线程占用的所有内存资源。

&emsp;&emsp;和 Linux 的 wait 函数不同，pthread_join 函数只能等待一个指定的线程终止。没有办法让 pthread_join 等待任意一个线程终止。

#### 分离线程

&emsp;&emsp;在任何一个时间点上，线程是可结合的或者是分离的。一个可结合的线程能够被其他线程回收和杀死，在被其他线程回收之前，它的内存资源 (例如栈) 是不释放的。相反，一个分离的线程是不能被其他线程回收或杀死的。它的内存资源在它终止时，由系统自动释放。

&emsp;&emsp;默认情况下，线程被创建成可结合的。为了避免内存泄漏，每个可结合线程都应该要么被其他线程显示回收，要么通过调用 pthread_detach 函数被分离。

```c
int pthread_detach(pthread_t tid);
```

&emsp;&emsp;pthread_detach 函数分离可结合线程 tid，线程可以通过以 pthread_self() 为参数分离自己。

#### 初始化线程

&emsp;&emsp;pthread_once 函数允许初始化与线程例程相关的状态。

```c
pthread_once_t once_control = PTHREAD_ONCE_INIT;

int pthread_once(pthread_once_t *once_control, void (*init_routine)(void));
```

&emsp;&emsp;once_control 是一个全局或者静态变量，并且总是被初始化为 PTHREAD_ONCE_INIT。当第一次调用 pthread_once 时，init_routine 会被调用执行。当 pthread_once 被第二次执行时，它会立即返回。这在多线程环境中保证初始化代码只被执行一次很有用。

### 并发服务器

&emsp;&emsp;基于线程的并发服务器代码很简单：

```c
#include "assist.h"

#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <sys/socket.h>

int main(int argc, char **argv)
{
    int listen_fd, *client_fd;
    socklen_t client_len;

    struct sockaddr_storage client_addr;
    char client_hostname[MAXLINE], client_port[MAXLINE];

    if (argc != 2) {
        fprintf(stderr, "usage: %s <port>\n", argv[0]);
        exit(0);
    }

    listen_fd = open_listen_fd(argv[1]);
    if (listen_fd == -1) {
        fprintf(stderr, "cannot create a listen socket\n");
        exit(1);
    }

    pthread_t tid;
    for (;;) {
        client_len = sizeof(client_addr);
        client_fd = malloc(sizeof(int));
        *client_fd = accept(listen_fd, (SA *)&client_addr, &client_len);
        pthread_create(&tid, NULL, thread, client_fd);
    }
}
```

&emsp;&emsp;thread 函数的内容也很简单：

```c
void *thread(void *argp)
{
    int client_fd = *((int *)argp);
    /* release resource */
    free(argp);
    pthread_detach(pthread_self());
    fprintf(stderr, "new connection\n");
    echo(client_fd);
    close(client_fd);
    return NULL;
}
```

## 多线程共享变量

&emsp;&emsp;使用一个示例程序来说明多线程程序中共享变量的问题：

```c
#define N (2)
void *thread(void *argp);

char **ptr;

int main()
{
    pthread_t tid;
    char *msg[N] = {
        "hello from foo",
        "hello from bar"
    };
    
    ptr = msgs;
    for (int i = 0; i < N; i++)
        pthread_create(&tid, NULL, thread, (void *)i);
    pthread_exit(NULL);
}

void *thread(void *argp)
{
    int myid = (int)argp;
    static int cnt = 0;
    printf("[%d]: %s (cnt=%d)\n", myid, ptr[myid], ++cnt);
	return NULL:
}
```

### 线程内存模型

&emsp;&emsp;一组并发线程运行在一个进程的上下文中。每个线程都有自己独立的线程上下文，包括线程 ID，栈，栈指针，程序计数器，条件码以及通用寄存器值。每个线程和其他线程一起共享进程上下文的剩余部分，包括整个用户虚拟地址空间。线程也共享相同的打开文件的集合。

&emsp;&emsp;从实际操作角度来说，一个线程读写另一个线程的寄存器值是不可能的。另一方面，因为所有线程共享虚拟内存的任意位置，如果某个线程修改了一个内存位置，那么其他每个线程最终都能在它读这个位置时发现这个变化。因此，寄存器是从不共享的，而虚拟内存总是共享的。

&emsp;&emsp;各自独立的线程栈内存模型不是那么整齐清晰，这些线程栈被保存在虚拟地址空间中的栈区域中，通常是被相应的线程独立的访问的。不同的线程栈是不对其他线程设防的，所以，如果一个线程以某种方式得到一个指向其他线程栈的指针，那么它就可以读写这个栈的任何部分。

### 变量映射内存

&emsp;&emsp;多线程的 C 程序中变量根据他们的存储类型被映射到内存：

- 全局变量：如变量 ptr。在运行时，全局变量只有一个实例，任何线程都能访问。
- 本地自动变量：如变量 myid。每个线程栈都包含它自己的所有本地自动变量的实例。
- 本地静态变量：如变量 cnt。虚拟内存中只包含一个本地静态变量的实例，每个线程都能访问。

### 共享变量

&emsp;&emsp;当且仅当变量 v 的一个实例被一个以上的线程引用时，变量 v 是共享的。如 ptr 和 cnt。

## 信号量同步线程

&emsp;&emsp;共享变量很方便，但是它也引入了**同步错误**的可能性：

```c
void *thread(void *argp);
volatile long cnt = 0;

int main()
{
    long niters;
    pthread_t tid1, tid2;
    
    niters = atoi(argv[1]);
    
    pthread_create(&tid1, NULL, thread, &niters);
    pthread_create(&tid2, NULL, thread, &niters);
    pthread_join(&tid1, NULL);
    pthread_join(&tid2, NULL);
    
    if (cnt != 2 * niters)
        printf("BOOM! cnt = %ld\n", cnt);
    else
        printf("OK cnt = %ld\n", cnt);
}

void *thread(void *argp)
{
    long niters = *((long *)argp);
    for (long i = 0; i < niters; i++)
        cnt++;
    return NULL;
}
```

&emsp;&emsp;经过多次运行发现，每次得到的结果都不同，并且都是错误的。观察 thread 中循环部分的汇编代码：

![](04.png)

&emsp;&emsp;可以将线程 i 的循环代码分解成五个部分：

- H~i~：在循环头部的指令块。
- L~i~：加载共享变量 cnt 到寄存器 %rdx~i~ (%rdx~i~ 表示线程 i 中的寄存器值)。
- U~i~：更新 %rdx~i~ 的指令。
- S~i~：将 %rdx~i~ 的更新值存回共享变量 cnt 的指令。
- T~i~：循环尾部的指令块。

&emsp;&emsp;当上面的程序的两个对等线程在一个单处理器上并发运行时，机器指令以某种顺序一个接一个的完成。因此，每个并发执行定义了两个线程中的指令的某种全序 (或者交叉)。这些顺序中的一些将会产生正确的结果，但是其他的则不会。

&emsp;&emsp;关键点在于：一般而言，没有办法预测操作系统是否为线程选择一个正确的顺序。

### 进度图

&emsp;&emsp;进度图将 n 个并发线程的执行模型化为一条 n 维笛卡尔空间中的轨迹线。每条轴 k 对应线程 k 的进度。每个点 (I~1~, I~2~, ..., I~n~) 代表线程已经完成了指令 I~k~ 这一状态。原点代表没有任何线程完成一条指令的初始状态。

![](05.png)

&emsp;&emsp;上图中水平轴对应线程 1，垂直轴对应线程 2。一个程序的执行历史被模型化为状态空间中的一条轨迹线：

![](06.png)

&emsp;&emsp;对于线程 i，操作共享变量 cnt 内容的指令 (L~i~, U~i~, S~i~) 构成了一个关于 cnt 的`临界区`，这个临界区不应该和其他线程的临界区交替执行。换句话说，必须确保每个线程在执行它的临界区中的指令时，拥有对共享变量的`互斥访问`。通常这种现象称为`互斥`。


![](07.png)

&emsp;&emsp;在进度图中，两个临界区的交集形成的状态空间区域称为`不安全区`。注意：不安全区和与它交界的状态紧连，但是不包括这些状态。绕开不安全区的轨迹线叫做`安全轨迹线`，相反，接触到任何不安全区的轨迹线叫做`不安全轨迹线`。

&emsp;&emsp;任何安全轨迹线都将正确的更新共享计数器。

### 信号量

&emsp;&emsp;信号量是一种经典的解决同步不同执行线程问题的方法。信号量 s 是具有非负整数值的全局变量，只能有两种特殊的操作来处理，这两种操作称为 P 和 V：

- P(s)：如果 s 非 0，那么 P 将 s 减 1，并立即返回。如果 s 为 0，那么就挂起这个线程，直到 s 变为 0，而一个 V 操作将会重启这个线程。在重启之后，P 操作将 s 减 1，并将控制返回调用者。
- V(s)：V 操作将 s 加 1。如果有任何线程阻塞在 P 操作等待 s 变成非 0，就会重启这些线程中的一个，然后该线程将 s 减 1，完成它的 P 操作。

&emsp;&emsp;P 中的测试和减 1 操作是不可分割的，也就是说：一旦检测到 s 变为非 0，就会将 s 减 1，不能有中断。V 中的加 1 操作也是不可分割的，也就是：加载，加 1，存储的过程没有中断。注意：V 的定义中没有定义等待线程被重启的顺序。因此，当有多个线程在等待同一个信号量时，不能预测 V 操作要重启哪个线程。

&emsp;&emsp;P 和 V 的定义确保一个正确初始化的信号量不会有负值，这个属性称为`信号不变性`。Posix 标准定义了许多操作信号量的函数：

```c
int sem_init(sem_t *sem, 0, unsigned int value);
int sem_wait(sem_t *s);  /* P(s) */
int sem_post(sem_t *s);  /* V(s) */
```

&emsp;&emsp;sem_init 函数将信号量 sem 初始化为 value。**每个信号量在使用前必须初始化。**

### 实现互斥

&emsp;&emsp;信号量提供了一种很方便的方法来确保对共享变量的互斥访问。基本思想是将每个共享变量 (或一组相关的共享变量) 与一个信号量 s (初始值为 1) 关联起来。然后用 P(s) 和 V(s) 操作将相应的临界区包围起来。

&emsp;&emsp;因为信号量的值总是 0 或者 1，所以这种信号量也叫做`二元信号量`，通常也称为`互斥锁`。在一个互斥锁上执行 P 操作称为加锁，执行 V 操作称为解锁。一个被用作一组可用资源的计数器的信号量称为`计数信号量`。

![](08.png)

&emsp;&emsp;上面标注了每个状态下信号量的值，P 和 V 的结合操作创建了一组禁止区 (s < 0) 禁止区包含不安全区，因为信号量的不可变性，没有实际可能的轨迹线能够接触到不安全区的任何部分。

&emsp;&emsp;利用互斥锁可以很轻松的解决之前的问题：

```c
volatile long cnt = 0;
sem_t mutex;

/* in main */
int main()
{
    sem_init(&mutex, 0, 1);
    /* ... */
}

/* in thread */
void *thread(void *argp)
{
    /* ... */
    for (long i = 0; i < niters; i++) {
        P(&mutex);
        cnt++;
        V(&mutex);
    }
}
```

### TODO 调度资源

&emsp;&emsp;待更新。

## 多线程服务器

&emsp;&emsp;所谓的预线程化其实就是线程池技术，线程池的主要思想就是在服务器启动时就创建一组工作线程，他们永不停息的工作。并且利用之前的生产者 - 消费者模型，主线程负责处理客户端连接，并将已连接描述符发送给线程池中的工作线程。

&emsp;&emsp;这么做的好处是：避免了每次有一个新客户端就要创建一个线程的开销，并且避免了短连接导致反复创建销毁线程的浪费，提高了服务器的吞吐量。

![](09.png)

&emsp;&emsp;使用 SBUF 包来实现一个线程池化的并发 echo 服务器。代码很简单，这里加入了一点新变动：echo_cnt 中包含了对所有客户端发送来的字节数的统计，虽然实际上它没什么用处，但是它使用了 pthread_once 进行初始化。

```c
#include "assist.h"

#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <pthread.h>
#include <sys/socket.h>

sbuf_t sbuf;

int main(int argc, char **argv)
{
    pthread_t tid;
    socklen_t client_len;
    int listen_fd, client_fd;
    struct sockaddr_storage client_addr;
    char client_hostname[MAXLINE], client_port[MAXLINE];

    if (argc != 2) {
        fprintf(stderr, "usage: %s <port>\n", argv[0]);
        exit(0);
    }

    listen_fd = open_listen_fd(argv[1]);
    if (listen_fd == -1) {
        fprintf(stderr, "cannot create a listen socket\n");
        exit(1);
    }

    sbuf_init(&sbuf, BUFSIZE);
    for (int i = 0; i < THREADS; i++)
        pthread_create(&tid, NULL, thread2, &sbuf);

    for (;;) {
        client_len = sizeof(client_addr);
        client_fd = accept(listen_fd, (SA *)&client_addr, &client_len);
        sbuf_insert(&sbuf, client_fd);
        fprintf(stderr, "new connection\n");
    }

    close(listen_fd);
}
```

&emsp;&emsp;echo_cnt 函数和 thread2 函数的内容：

```c
static int byte_cnt;
static sem_t mutex;

static void init_echo_cnt(void)
{
    sem_init(&mutex, 0, 1);
    byte_cnt = 0;
}

void echo_cnt(int client_fd)
{
    int nread;
    char buf[MAXLINE];

    static pthread_once_t once = PTHREAD_ONCE_INIT;
    pthread_once(&once, init_echo_cnt);

    while ((nread = read(client_fd, buf, MAXLINE - 1)) > 0) {
        buf[nread] = 0;

        sem_wait(&mutex);
        byte_cnt += nread;
        fprintf(stderr, "from client: %s (%d / %d)", buf, nread, byte_cnt);
        sem_post(&mutex);

        /* write back to client */
        write(client_fd, buf, nread);
    }
}

void *thread2(void *sbuf)
{
    pthread_detach(pthread_self());
    for (;;) {
        int fd = sbuf_remove((sbuf_t *)sbuf);
        echo_cnt(fd);
        close(fd);
    }
}
```

## TODO 线程提高并行

&emsp;&emsp;之前所有的例子都假设并发线程是在单处理器系统上执行的，然而，大多数现代机器具有多核处理器。并发程序通常在这样的机器上运行的更快，因为操作系统内核会在多个核上并行的调度这些并发线程。

![](10.png)

&emsp;&emsp;将任务分配到不同的线程最直接的方法是将序列划分成 t 个不相交的区域，然后给 t 个不同的线程每个分配一个区域。一个简单的例子是求和一列整数 0, 1 ..., n - 1。最后赋值给全局变量 gsum。

&emsp;&emsp;待更新。

### 并行程序性能

&emsp;&emsp;之前 psum-local 的运行时间如图，它是一个关于线程数量的函数：随着线程数的增加，运行时间下降，知道增加到四个线程 (运行在四核处理器上)，此时，运行时间趋于平稳，甚至开始有点下降。

![](11.png)

&emsp;&emsp;通常，使用`加速比`来衡量并行程序的性能：S~p~ = T~1~ / T~p~ (这个公式也被称为`强扩展`)。其中 p 是处理器核心数量，T~p~ 是程序在 p 个核心上的运行时间，T~1~ 是顺序程序的运行时间。S~p~ 称为`绝对加速比`。当 T~1~ 是并行版本程序在一个核心上的运行时间时，S~p~ 称为`相对加速比`。

&emsp;&emsp;一种相关的测量量称为`效率`：E~p~ = S~p~ / p。效率范围通常在 (0, 100]，效率用于测量由并行化造成的开销。具有高效率的程序比低效率的程序花费更多时间在有用的工作上。换句话说，低效率的程序在同步和通信上开销更大。

![](12.png)

## 其他并发问题

&emsp;&emsp;一旦要求同步对共享数据的访问，那么事情就变得复杂起来。同步从根本上说是很难的问题，它引出了在普通的顺序程序中完全不会出现的问题。

### 线程安全

&emsp;&emsp;一个函数被称为`线程安全`的，当且仅当被多个并发线程反复的调用时，它会一直产生正确的结果。如果一个函数不是线程安全的，就说它是`线程不安全`的。

&emsp;&emsp;可以定义四类不相交的线程不安全函数类：

1. 不保护共享变量的函数

&emsp;&emsp;在信号量同步线程开始时写到的 thread 函数就是这类，该函数对一个未受保护的全局计数器变量加 1。将这类函数变成线程安全的，相对而言比较简单：利用像 P V 操作这样的同步操作来保护共享变量。这个方法的优点是在调用程序中不需要做任何修改，缺点是同步操作将减慢程序的执行时间。

2. 保持跨越多个调用的状态的函数

&emsp;&emsp;一个伪随机数生成器是这类线程不安全函数的例子：

```c
unsigned next_seed = 1;

unsigned rand(void)
{
    next_seed = next_seed * 1103515245 + 12543;
    return (unsigned)(next_seed >> 16) % 32768;
}

void srand(unsigned new_seed)
{
    next_seed = new_seed;
}
```

&emsp;&emsp;rand 函数是线程不安全的，因为当前调用的结果依赖于前两次调用的中间结果。简单来说：当调用 srand 设置了一个种子后，在一个单线程中反复调用 rand，就能得到一个可重复的随机数字序列。然而，如果多线程调用 rand 函数，这种假设就不成立了。

&emsp;&emsp;使得像 rand 这样的函数线程安全的唯一方式是重写它，使得它不再使用任何 static 数据，而是依靠调用者在参数中传递状态信息。这样做的缺点是，程序员现在被迫要修改调用程序中的代码。

3. 返回指向静态变量指针的函数

&emsp;&emsp;例如 ctime 和 gethostbyname 函数，他们将计算结果放在一个 static 变量中，然后返回一个指向这个变量的指针。如果从并发线程中调用这些函数，那么可能会发生数据覆盖。

&emsp;&emsp;解决办法有两种：一种是选择重写这类函数，让调用者传递存放结果的地址。这就消除了共享数据，但是这种办法要求程序员能够修改函数的源代码。

&emsp;&emsp;另一种方法是使用加锁 - 复制技术。基本思想是将线程不安全的函数与互斥锁联系起来：在每一个调用位置，对互斥锁加锁，调用线程不安全的函数，将函数返回的结果复制到一个私有的内存位置，然后对互斥锁解锁。为了减少对调用者的修改，应该定义一个线程安全的包装函数，它执行加锁 - 复制，然后通过调用这个包装函数来取代所有对线程不安全函数的调用，一个 ctime 的线程安全包装如下：

```c
char *ctime_ts(const time_t *timep, char *privatep)
{
    char *sharedp;
    
    P(&mutex);
    sharedp = ctime(timep);
    strcpy(privatep, sharedp);
    V(&mutex);
    
    return privatep;
}
```

4. 调用线程不安全函数的函数

&emsp;&emsp;如果函数 f 调用线程不安全函数 g，有两种情况：如果 g 是第 2 类线程不安全函数，那么除了重写 g 以外没什么方法，所以 f 也是线程不安全的函数。如果 g 是第 1 或 3 类线程不安全函数，那么只要用一个互斥锁保护调用位置和得到的任何共享数据，f 仍可能是一个线程安全的函数。

### 可重入性

&emsp;&emsp;可重入函数是一类重要的线程安全的函数，它不依赖于外部状态,因此可以在同一个线程的多个调用栈中安全使用。一个重要特点是：可重入函数不引用任何共享数据。

![](13.png)

&emsp;&emsp;通常来说，可重入函数要比不可重入的线程安全函数更高效，因为他们不引用外部共享数据，所以不需要做同步操作。一个可重入版本的 rand 函数：

```c
int rand_r(unsigned int *nextp)
{
    *nextp = *nextp + 1103515245 + 12345;
    return (unsigned int)(*nextp / 65535) % 32768;
}
```

&emsp;&emsp;可重入函数有两类，都统称为可重入函数：显示可重入的和隐式可重入的。显示可重入函数要求函数参数都是传值传递的 (没有指针)，并且所有数据引用都是本地的自动栈变量 (不引用静态或者全局变量)。隐式可重入函数在显示的基础上放松了一点要求：允许函数参数中一些参数是引用传递的 (允许指针)，如果调用线程小心的传递指向非共享数据的指针，那么它是可重入的。

### 使用库函数

&emsp;&emsp;大多数 Linux 函数，包括定义在标准 C 中的函数 (malloc, free, realloc, printf, scanf...) 都是线程安全的，只有一小部分是例外 (只列一部分)：

![](14.png)

&emsp;&emsp;除了 rand 和 strtok 以外，其他的都是第 3 类线程不安全函数，他们返回一个静态变量的指针。可以使用加锁 - 复制技术写一个包装函数，但是这会因为同步拖慢程序的速度。但是加锁 - 复制技术对 rand 这类第 2 类线程不安全函数并无效果。

&emsp;&emsp;因此 Linux 提供大多数线程不安全函数的可重入版本，可重入版本总是以 _r 后缀结尾。

### 竞争

&emsp;&emsp;当一个程序的正确性依赖于一个线程要在另一个线程到达 y 点之前到达它的控制流中的 x 点时，就会发生竞争。一个简单的例子：

```c
#define N (4)
void *thread(void *vargp);

int main()
{
    pthread_t tid[N];
    for (int i = 0; i < N; i++)
        pthread_create(&tid[i], NULL, thread, &i);
    for (int i = 0; i < N; i++)
        pthread_join(tid[i], NULL);
}

void *thread(void *vargp)
{
    int myid = *((int *)vargp);
    printf("Hello from thread %d\n", myid);
    return NULL;
}
```

&emsp;&emsp;这个程序正确的前提是：对等线程执行第 15 行的时机必须在主线程创建线程后执行第 8 行更新 i 之前。问题在于，这个时机是由内核调度决定的，所以这个程序发生了竞争。一个解决办法是使用动态分配为每个线程分配一个独立的 id，但是这些块必须及时回收：
```c
int main()
{
    /* ... */
    for (int i = 0; i < N; i++) {
        int *ptr = malloc(sizeof(int));
        *ptr = i;
        pthread_create(&tid[N], NULL, thread, ptr);
    }
    /* ... */
}

void *thread(void *vargp)
{
    int myid = *((int *)vargp);
    free(vargp);
    /* ... */
}
```

### 死锁

&emsp;&emsp;死锁是信号量引入的一种潜在的运行时错误。它指一组线程被阻塞了，等待一个永远也不会为真的条件。

![](15.png)

- 由于 P V 操作顺序不当，两个信号量的禁止区域重叠。如果某个轨迹线到达了死锁状态 d，那么就不可能有进一步的进展了。因为重叠的禁止区域阻塞了各个合法方向上的进展。
- 重叠的禁止区域引起了一组称为死锁区域的状态，如果一个轨迹线到达了死锁区域，那么它将不可能离开。
- 死锁困难之处在于它是不可预测的，也许前 1000 次程序都运行正常，但是下一次就发生死锁。

&emsp;&emsp;当使用二元信号量来实现互斥时，可以使用一种简单有效的规则避免死锁：

&emsp;&emsp;互斥加锁顺序规则：给定所有互斥操作的一个全序，如果每个线程都是以一种顺序获得互斥锁并且以相反的顺序释放，那么这个程序就是无死锁的。

![](16.png)

&emsp;&emsp;19:30 Nov 15 2023.
