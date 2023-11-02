---
title: CSAPP 11 - 网络编程
date: 2023-08-23 12:12:25
updated: 2023-08-26 19:00:00
tags:
  - CSAPP
  - C
categories:
  - CSAPP
---

&emsp;&emsp;网络应用由一个服务器进程和一个或者多个客户端进程组成。

<!-- more -->

## 网络模型

&emsp;&emsp;每个网络应用都是基于**客户端 - 服务器**模型的。服务器管理某种资源，并且通过操作这种资源来为他的客户端提供某种服务。客户端 - 服务器模型中的基本操作是`事务`，一个事务由以下四步组成：

1. 当客户端需要服务时，它向服务器发送一个请求，发起一个事务。
2. 服务器收到请求后，解决它，并以适当的方式操作他的资源。
3. 服务器给客户端发送一个响应，并等待下一个请求。
4. 客户端接受到响应并且处理它。

![](01.png)

&emsp;&emsp;很重要的一点是：客户端和服务器是进程而不是主机。一台主机可以同时运行多个不同的客户端和服务器。并且，客户端 - 服务器事务不是数据库事务。

## 网络

&emsp;&emsp;客户端和服务器通常运行在不同的主机上，并且通过**计算机网络**的硬件和软件资源来通信。对于主机而言，网络只是又一种 I/O 设备，是数据源和数据接收方。

&emsp;&emsp;一个插到 I/O 总线扩展槽的适配器提供了到网络的物理接口，从网络上接收到的数据从适配器经过 I/O 和内存总线复制到内存，通常是通过 DMA 传送。相似的，数据也能从内存复制到网络。

![](02.png)

&emsp;&emsp;物理上而言，网络是一个按照地理远近组成的层次系统。最低层是 LAN 局域网，在一个建筑或者校园范围内。迄今为止，最流行的局域网技术是`以太网`。

&emsp;&emsp;一个`以太网段`包括一些电缆和集线器。以太网通常跨越一些小的区域，每根电缆都有相同的最大位宽，通常是 100 Mb/s 或者 1 Gb/s。一端链接到主机的适配器，另一端链接到集线器的一个`端口`上。集线器不加分辨的将从一个端口接收到的所有位复制到其他端口上。因此，每台主机都能看到每个位。

&emsp;&emsp;每个以太网适配器都有一个全球唯一的 48 位地址 (MAC 地址)，它存储在这个适配器的 ROM 上。一台主机可以发送一段位 (帧) 到这个网段内的其他任何主机。每个帧包含一些固定数量的`头部`位，用来标识此帧的源和目的地址以及此帧的长度，此后紧随的是数据位的`有效载荷`。每个主机都能看到这个帧，但是只有目的主机实际读取它。

&emsp;&emsp;使用一些电缆和`网桥`，多个以太网段可以链接成较大的局域网，称为`桥接以太网`。在一个桥接以太网中，一些电缆链接网桥与网桥，而另一些链接网桥与集线器。这些电缆的带宽可以是不同的：

![](03.png)

&emsp;&emsp;网桥比集线器更加充分的利用了电缆带宽，利用分配算法，他们随着时间自动学习哪个主机可以通过哪个端口可达，然后只在有必要时，有选择的将帧从一个端口复制到另一个端口。为了简化局域网的表示，将集线器和网桥以及链接他们的电缆画成一根水平线。

&emsp;&emsp;在层次的更高级别中，多个**不兼容**的局域网可以通过叫做`路由器`的特殊计算机链接起来，组成一个互联网络。每台路由器对于它所链接到的每个网络都有一个适配器 (端口)。路由器也能链接高速点到点电话链接，称为 WAN 广域网。

![](04.png)

&emsp;&emsp;互联网络的一个关键特性是：它可以由使用完全不同且不兼容技术的局域网和广域网组成。因此，必须在每台主机和路由器上运行**协议软件**来消除不同网络之间的差异。该软件实现的协议将管理主机和路由器如何协作以传输数据，它提供以下两个基本功能：

- 命名机制：为主机地址定义统一的格式，并为每台主机分配至少一个唯一标识它的互联网地址 (IP 地址)。
- 传送机制：定义一种统一的方式将数据位封装为若干个不连续的块，即数据包。一个包是由**包头**和**有效载荷**组成的，包头中包含了包的大小以及源主机和目的主机的地址，有效载荷包括从源主机发送的数据位。

&emsp;&emsp;一个利用互联网络协议使主机和路由器在两个不兼容的局域网之间通信的示例：

![](05.png)

## 因特网

&emsp;&emsp;全球 IP 因特网是最著名和最成功的互联网络实现：

![](06.png)

&emsp;&emsp;每台因特网主机都运行实现了 TCP/IP 协议的软件，几乎每个现代计算机系统都支持这个协议。因特网的客户端和服务器混合使用**套接字接口函数**和 **Unix I/O** 函数来进行通信。通常将套接字接口函数实现为系统调用，这些系统调用会陷入内核，并调用内核模式的 TCP/IP 函数。

&emsp;&emsp;TCP/IP 是一个协议族，其中每一个都提供不同的功能。IP 协议提供基本的命名方法和递送机制，这种递送机制能从一台主机向其他主机发送包 (数据报)。IP 协议从某种意义上而言是不可靠的，因为，如果数据报在网络中丢失或者重复，它并不会尝试恢复。UDP 稍微扩展了 IP 协议，使得包可以在进程间而不是主机间传递。TCP 是建立在 IP 之上的复杂协议，提供了进程间可靠的`全双工`链接。

&emsp;&emsp;从程序员角度，可以将因特网看作一个世界范围的主机集合，满足以下特性：

- 主机集合被映射为一组 32 位的 IP 地址。
- 这组 IP 地址被映射为一组因特网域名。
- 因特网主机上的进程能通过链接和任何其他因特网主机上的进程通信。

### IP 地址

&emsp;&emsp;一个 IP 地址就是一个 32 位无符号整数，网络程序将 IP 地址存放在 IP 地址结构中：

```c
struct in_addr {
    uint32_t s_addr;
};
```

&emsp;&emsp;因为因特网主机可以有不同的主机字节顺序，TCP/IP 为任意整数数据项定义了统一的`网络字节序`即大端序。IP 地址放在包头中跨过网络被携带，Unix 提供了以下函数在网络和主机字节顺序间实现转换：

```c
uint32_t htonl(uint32_t hostlong);
uint16_t htons(uint16_t hostshort);

uint32_t ntohl(uint32_t netlong);
uint16_t ntohs(uint16_t netshort);
```

&emsp;&emsp;htonl 函数将 32 位整数由主机字节顺序转换为网络字节序，ntohl 正好相反。没有处理 64 位值的函数。

&emsp;&emsp;IP 地址通常是以**点分十进制**表示法来表示的，这里，每个字节由他的十进制值表示，并且用句点和其他字节分开。在 Linux 系统上，可以使用 hostname 命令来确定自己主机的点分十进制地址。

```bash
hostname -i
```

&emsp;&emsp;应用程序使用 inet_pton 和 inet_ntop 函数来实现 IP 地址和点分十进制字符串之间的转换：

```c
int inet_pton(AF_INET, const char *src, void *dest);

const char *inet_ntop(AF_INET, const char *src, char *dest, socklen_t size);
```

### 因特网域名

&emsp;&emsp;因特网客户端和服务器互相通信使用的是 IP 地址，然而大整数很难人工记忆，所以因特网也提供了一组`域名`，以及一种将域名映射到 IP 地址的机制。域名是一串用句点分割的单词。

&emsp;&emsp;域名集合形成了一个层次结构，每个域名编码了它在这个层次中的位置：

![](07.png)

&emsp;&emsp;因特网定义了域名集合和 IP 地址集合之间的映射。直到 1988 年，这个映射都是通过 HOST.txt 的文本文件来手工维护的。从那以后，这个映射是通过分布在世界范围内的数据库 DNS 来维护的。

### 因特网连接

&emsp;&emsp;因特网客户端和服务器通过在**连接**上发送和接收字节流来通信。从连接一对进程的角度而言，连接是点对点的。从数据可以同时双向流动的角度而言，它是全双工的。并且从由源进程发出的字节流最终被目的进程以它发出的顺序收到它的角度而言，它也是可靠的。

&emsp;&emsp;一个套接字是连接的一个端点，每个套接字都有相应的套接字地址，是由一个因特网地址和一个 16 位整数端口组成的，用 address:port 表示。

&emsp;&emsp;当客户端发起一个连接请求时，客户端套接字地址中的端口是由内核自动分配的，称为**临时端口**。但是，服务器套接字地址中的端口通常是某个**知名端口**。一个连接是由它两端的套接字地址唯一确定的，这对套接字地址叫做**套接字对**，可以用元组 (cliaddr:port, servaddr:port) 来表示。

![](08.png)

## 套接字接口

&emsp;&emsp;套接字接口是一组函数，他们和 Unix I/O 函数结合起来，用以创建网络应用。大多数现代系统都实现了套接字接口，一个典型的客户端 - 服务器事务的上下文中的套接字接口概述：

![](09.png)

### 套接字地址

&emsp;&emsp;从 Linux 内核的角度来看，一个套接字就是通信的一个端点。从 Linux 程序的角度来看，套接字就是一个有相应描述符的打开文件。

&emsp;&emsp;因特网的套接字地址存放在一个类型位 sockaddr_in 的 16 字节结构体中。对于因特网应用，sin_family 成员是 AF_INET，sin_port 成员是一个 16 位的端口号，sin_addr 成员就是一个 32 位的 IP 地址。注意：IP 地址和端口号总是以网络字节序存放：

```c
/* IP socket address structure */
struct sockaddr_in  {
    uint16_t        sin_family;  /* Protocol family (always AF_INET) */
    uint16_t        sin_port;    /* Port number in network byte order */
    struct in_addr  sin_addr;    /* IP address in network byte order */
    unsigned char   sin_zero[8]; /* Pad to sizeof(struct sockaddr) */
};
```

&emsp;&emsp;在调用函数 connect bind 和 accept 时，需要传入一个指向套接字地址结构体的指针。由于套接字有多种类型，不同协议的套接字地址结构体类型也有所不同。IPv6 套接字地址存储在 sockaddr_in6 类型的结构体中，sin_family 字段为 AF_INET6。Unix Domain 套接字地址存储在 sockaddr_un 类型的结构体中，sin_family 字段为 AF_UNIX。在套接字接口设计者所处的时代，C 还并不支持使用 void* 指针。于是他们只好重新定义一个适用于所有协议 sockaddr 结构体，然后要求应用程序将任何与协议有关的结构体指针转换为这种通用的结构体指针：

```c
/* Generic socket address structure (for connect, bind, and accept) */
struct sockaddr {
    uint16_t  sa_family;    /* Protocol family */
    char      sa_data[14];  /* Address data  */
};
```

### socket 函数

&emsp;&emsp;客户端和服务器使用 socket 函数来创建一个套接字描述符：

```c
int socket(int domain, int type, int protocol);
```

&emsp;&emsp;如果希望套接字成为连接的端点，可以使用以下参数调用该函数：

```c
clientfd = socket(AF_INET, SOCK_STREAM, 0);
```

&emsp;&emsp;其中，AF_INET 代表使用 32 位 IP 地址，SOCK_STREAM 表示套接字将成为连接的端点。该函数返回的描述符 clientfd 只是部分打开，还不能进行读写。

### connect 函数

&emsp;&emsp;客户端通过调用 connect 函数来建立和服务器的连接：

```c
int connect(int clientfd, const struct sockaddr *addr, socklen_t addrlen);
```

&emsp;&emsp;connect 函数试图与套接字地址为 addr 的服务器建立一个因特网连接，其中 addrlen 是 sizeof(sockaddr_in)。connect 函数会阻塞，一直到连接成功建立或者是发生错误。如果成功，clientfd 描述符现在就准备好可以读写了，并且得到了套接字对 (x:y, addr.sin_addr:addr:sin_port)，其中 x 是客户端 IP 地址，y 是内核分配的临时端口。该套接字对唯一确定了客户端和服务端进程。

### bind 函数

&emsp;&emsp;bind 函数请求内核将参数 addr 中的服务器套接字地址与套接字描述符 sockfd 关联起来，参数 addrlen 是结构体 sockaddr_in 的大小：

```c
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

### listen 函数

&emsp;&emsp;客户端是发起连接请求的主动实体，服务器是等待来自客户端连接请求的被动实体。默认情况下，内核会认为 socket 函数创建的描述符对应于**主动套接字**，它存在于一个连接的客户端。服务器调用 listen 函数告诉内核，描述符是被服务器而不是客户端使用的。

```c
int listen(int sockfd, int backlog);
```

&emsp;&emsp;listen 函数将 sockfd 从一个主动套接字转化为一个**监听套接字**，该套接字可以接受来自客户端的连接请求。backlog 指示了等待连接请求队列的最大长度。

### accept 函数

&emsp;&emsp;服务器通过调用 accept 函数来等待来自客户端的连接请求：

```c
int accept(int listenfd, struct sockaddr *addr, int *addrlen);
```

&emsp;&emsp;accept 函数等待来自客户端的连接请求到达侦听描述符 listenfd，然后在 addr 中填写客户端的套接字地址，并返回一个**已连接描述符**，这个描述符可被用来利用 Unix I/O 函数与客户端通信。

&emsp;&emsp;监听描述符通常只被创建一次，并存在于服务器的整个生命周期。已连接描述符是客户端和服务器之间已经建立起来的连接的一个端点，它只存在于服务器为一个客户端服务的过程。

![](10.png)

### 主机和服务转换

&emsp;&emsp;Linux 提供了一些强大的函数实现二进制套接字地址结构和主机名，主机地址，服务名和端口号的字符串表示之间的转化。可以配合与套接字接口一起使用，编写独立于任何特定版本的 IP 协议的网络程序：

1. getaddrinfo 函数

&emsp;&emsp;getaddrinfo 函数将主机名 (或主机地址) 和服务名 (或端口号) 转换为套接字地址结构体：

```c
struct addrinfo {
    int     ai_flags;           /* Hints argument flags */
    int     ai_family;          /* First arg to socket function */
    int     ai_socktype;        /* Second arg to socket function */
    char    ai_protocol;        /* Third arg to socket function  */
    char    *ai_canonname;      /* Canonical hostname */
    size_t  ai_addrlen;         /* Size of ai_addr struct */
    struct  sockaddr *ai_addr;  /* Ptr to socket address structure */
    struct  addrinfo *ai_next;  /* Ptr to next item in linked list */      
}

int getaddrinfo(const char *host, const char *service,
                const struct addrinfo *hints,
                struct addrinfo **result);
```

&emsp;&emsp;该函数会根据 hints 指定的规范分配并初始化一个 addrinfo 结构体链表，其中每个结构体的 ai_addr 字段都指向一个与 host 和 service 对应的套接字地址，result 指向链表头部：

![](11.png)

&emsp;&emsp;参数 host 可以是域名，也可以是数字地址 (如点分十进制 IP 地址)。参数 service 可以是服务名称 (如 http)，也可以是十进制端口号。如果不需要套接字地址中的主机名，就可以将 host 设为 NULL。对于服务名来说也是如此，不过两者不能同时为 NULL。

&emsp;&emsp;客户端在调用该函数后会遍历上述链表，依次使用每个套接字地址作为参数调用 socket 和 connect 函數直至成功并建立连接。服务器在调用该函数后会遍历上述链表，依次使用每个套接字地址作为参数调用 socket 和 bind 直至成功且描述符被绑定到一个有效的套接字地址。

&emsp;&emsp;getaddrinfo 会为同一个 host 和 service 初始化多个 addrinfo 结构体，这是因为：主机可能是多宿主的，可以通过多种协议 (如 IPv4 和 IPv6) 访问。客户端可以通过不同的套接字类型 (如 SOCK_STREAM 和 SOCK_DGRAM) 访问相同的服务。因此通常会根据需求设置 hints 参数，以使函数生成期望的套接字地址。

&emsp;&emsp;当 hints 作为参数传递时，只有 ai_family，ai_socktype，ai_protocol 和 ai_flags 字段可以被设置，其他字段必须为 0 或 NULL。在实际使用中，会使用 memset 函数将 hints 归零，然后设置以下字段：

- ai_family 为 AF_INET 时，该函数将生成 IPv4 套接字地址。ai_family 为 AF_INET6 时，该函数将生成 IPv6 套接字地址。

- 对于面向连接的网络应用程序，ai_socktype 应当设为 SOCK_STREAM。

- ai_flags 是能够修改函数默认行为的位掩码，主要包括：

  - AI_ADDRCONFIG：仅当本地主机使用 IPv4 时生成 IPv4 Socket 地址。

  - AI_CANONNAME：默认情况下，addrinfo 结构体内的 ai_canonname 字段为 NULL。若设置该掩码，函数会将链表中第一个 addrinfo 结构体内的 ai_canonname 字段指向主机的规范名称。

  - AI_NUMERICSERV：强制参数 service 使用端口号。

  - AI_PASSIVE：服务器可以使用该函数生成的套接字地址创建监听描述符。在这种情况下，参数 host 应当设为 NULL，表示服务器的所有 IP 地址均可用于连接 (即 INADDR_ANY 或 0.0.0.0)。

&emsp;&emsp;当 getaddrinfo 初始化 addrinfo 结构体链表时，它会填充除 ai_flags 之外的所有字段。ai_family，ai_socktype 和 ai_protocol 可以直接传递给 socket 函数，ai_addr 和 ai_addrlen 可以直接传递给 connect 和 bind 函数。因此能够使用它编写适用于任何版本 IP 协议的客户端和服务器。

&emsp;&emsp;为了避免内存泄漏，应用程序最终必须调用 freeaddrinfo 函数释放链表：

```c
void freeaddrinfo(struct addrinfo *result);
```

&emsp;&emsp;getaddrinfo 函数会返回非零错误码，应用程序可以调用 gai_strerror 函数将其转换为消息字符串：

```c
const char *gai_strerror(int errcode);
```

2. getnameinfo 函数

&emsp;&emsp;getnameinfo 函数功能和 gethostinfo 正好相反，它可以将一个套接字地址结构转化为相应的主机和服务名字符串，并且它是可重入和与协议无关的：

```c
int getnameinfo(const struct sockaddr *sa, socklen_t salen,
                char *host, size_t hostlen,
                char *service, size_t servlen,
                int flags);
```

&emsp;&emsp;参数 sa 指向一个大小为 salen 字节的套接字地址结构体，host 指向一个大小为 hostlen 字节的缓冲区，而 service 则指向一个大小为 servlen 字节的缓冲区。该函数将 sa 转换为主机名和服务名字符串，然后将它们复制到 host 和 service 指向的缓冲区。如果该函数返回非零错误代码，应用程序可以调用 gai_strerror 将其转换为消息字符串。如果不需要主机名，就可以将 host 设为 NULL。对于服务名来说也是如此，不过两者不能同时为NULL。

&emsp;&emsp;参数 flags 是修改函数默认行为的位掩码，包括：

- NI_NUMERICHOST：默认情况下，函数将在 host 指向的缓冲区中生成一个域名。若设置该掩码，函数会生成一个数字地址字符串。
- NI_NUMERICSERV：默认情况下，函数将在 etc/services 件中查找并生成服务名。若设置该掩码，函数会跳过查找并生成端口号。

&emsp;&emsp;TODO 示例代码：

&emsp;&emsp;待更新。

### TODO 辅助函数

&emsp;&emsp;待更新。

### echo 服务

&emsp;&emsp;下面是一个简单的 echo 客户端和服务器示例。客户端在和服务器建立链接后，会进入循环状态，反复从标准输入中读取文本行，然后将文本行发送给服务器，然后从服务器读取回送的行并输出到标准输出中。当从标准输入中读取到 EOF 时就结束循环，并关闭套接字。

```c
#include "assist.h"

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

int main(int argc, char **argv)
{
    int client_fd;
    char *host, *port, buf[MAXLINE] = {0};

    if (argc != 3) {
        fprintf(stderr, "usage: %s <host> <port>\n", argv[0]);
        exit(0);
    }

    host = argv[1];
    port = argv[2];

    client_fd = open_client_fd(host, port);
    if (client_fd == -1) {
        fprintf(stderr, "cannot connect to %s:%s\n", host, port);
        exit(1);
    }

    while (fgets(buf, MAXLINE, stdin) != NULL) {
        write(client_fd, buf, MAXLINE);
        memset(buf, 0, MAXLINE);

        if (read(client_fd, buf, MAXLINE) < 1)
            break;

        fprintf(stderr, "from server: ");
        write(2, buf, MAXLINE);
        memset(buf, 0, MAXLINE);
    }

    close(client_fd);
}
```

&emsp;&emsp;当客户端关闭套接字描述符后，一个 EOF 会被发送到服务器。服务器从它的 read 返回值中接收到 0，也会关闭这个客户端描述符。echo 服务器首先打开监听描述符，然后进入无限循环，每次循环都等待一个客户端链接，并调用 echo 函数为这个客户端服务。

```c
#include "assist.h"

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>

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

    listen_fd = open_listen_fd(argv[1]);
    if (listen_fd == -1) {
        fprintf(stderr, "cannot create a listen socket\n");
        exit(1);
    }

    for (;;) {
        client_len = sizeof(client_addr);
        client_fd = accept(listen_fd, (struct sockaddr *)&client_addr, &client_len);
        fprintf(stderr, "new connection\n");
        echo(client_fd);
        close(client_fd);
    }
}
```

&emsp;&emsp;echo 函数设计很简单，因为数据在客户端被格式化好后发送过来，所以服务器可以直接使用这些数据。

```c
void echo(int client_fd)
{
    char buf[MAXLINE];

    while (read(client_fd, buf, MAXLINE) > 0) {
        fprintf(stderr, "from client: ");
        write(2, buf, MAXLINE);

        // write back to client
        write(client_fd, buf, MAXLINE);
    }
}
```

## Web 服务器

&emsp;&emsp;在之前 echo 服务器的基础上，利用网络编程的基本概念来构建一个小但是功能齐全的 Web 服务器。

### Web 基础

&emsp;&emsp;Web 客户端和服务器之间的交互用的是一个基于文本的应用级协议即 HTTP 协议。HTTP 是一个简单的协议，一个 Web 客户端 (浏览器) 打开一个到服务器的因特网连接，并且请求某些内容。服务器响应所请求的内容，然后关闭连接。浏览器读取这些内容，并把它显示在屏幕上。

&emsp;&emsp;Web 服务和常规的 FTP 区别主要在于：Web 内容可以使用 HTML 语言编写，它告诉浏览器如何显示这页中的各种文本和图形对象。

### Web 内容

&emsp;&emsp;对于 Web 客户端和服务器而言，内容是一个 MIME 类型相关的字节序列，一些常用的 MIME 类型：

![](12.png)

&emsp;&emsp;Web 服务器通过两种不同的方式向客户端提供内容：

1. 服务静态内容：取一个磁盘文件 (静态内容)，并将磁盘文件的内容返回给客户端。
2. 服务动态内容：运行一个可执行文件，并将他的输出返回给客户端。

&emsp;&emsp;每条 Web 服务器返回的内容都是和他管理的某个文件相关联的。这些文件中的每一个都有一个唯一的名字，叫做 URL，例如 URL：

```
http://www.google.com:80/index.html

http://bluefish.ics.cs.cmu.edu:8000/cgi-bin/adder?15000&213
```

&emsp;&emsp;端口号是可选的，默认为知名的 HTTP 端口 80，第一条 URL 表示主机上一个称为 index 的 html 文件，它由监听端口为 80 的 Web 进程管理。第二条 URL 关联的文件是 cgi-bin/adder，它由端口号为 8000 的 Web 进程管理。并且 ? 字符分隔了文件名和参数，每个参数用 & 分隔。

### HTTP 事务

&emsp;&emsp;HTTP 是基于在因特网连接上传送文本行的，可以使用 telnet 程序来和因特网上的任何 Web 服务器执行事务。

```bash
telnet www.aol.com:80
```

1. HTTP 请求

&emsp;&emsp;一个 HTTP 请求组成包括：一个请求行，后面跟随零个或多个请求报头，再跟随一个空的文本行来终止报头列表。请求行的形式是：

```
method URI version
```

&emsp;&emsp;HTTP 支持许多不同的方法，包括 GET POST OPTIONS HEAD PUT DELETE 和 TRACE。常用的就是 GET 请求方法，GET 方法指导服务器生成和返回 URI 标识的内容，URI 就是相应 URL 的后缀，包括文件名和可选的参数。

&emsp;&emsp;请求行中的 version 字段表明了该请求遵循的 HTTP 版本。最新的 HTTP 版本是 HTTP/1.1，它在 1.0 的基础上增加了一些新的特性。在实际上，两个版本是相互兼容的。

&emsp;&emsp;请求报头为服务器提供了额外的信息，例如浏览器的商标名，或者浏览器理解的 MIME 类型。请求报头的形式为：

```
header-name: header-data
```

2. HTTP 响应

&emsp;&emsp;HTTP 响应和 HTTP 请求是类似的。一个 HTTP 响应的组成包括：一个响应行，后面跟随着零个或多个响应报头，再跟随一个终止报头的空行，再跟随一个响应主体。一个响应行的格式是：

```
version status-code status-message
```

&emsp;&emsp;version 字段描述的是响应所遵循的 HTTP 版本。状态码是一个 3 位的正整数，指明对请求的处理。状态消息给出与错误代码等价的英文描述。一些常见的状态码以及对应的消息：

![](13.png)

&emsp;&emsp;响应报头提供了关于响应的附加消息，常见的如 Content-Type，它告诉客户端响应主体内容的 MIME 类型，Content-Length，它指示了响应主体的字节大小。响应主体和响应报头之间需要使用一个空行隔开。

### 服务动态内容

&emsp;&emsp;服务器向客户端提供动态内容可以使用 CGI 标准来实现。

1. 客户端将参数传递给服务器

&emsp;&emsp;GET 请求的参数在 URI 中传递。? 分隔了文件名和参数列表，每个参数使用 & 分隔。参数中不能有空格，必须用字符 %20 表示，其他特殊字符类似。

&emsp;&emsp;POST 请求中，参数在请求主体而不是 URI 中。

2. 服务器将参数传递给子进程

&emsp;&emsp;如果服务器接收到一个请求：

```
GET /cgi-bin/adder?15000&213 HTTP/1.1
```

&emsp;&emsp;它调用 fork 创建一个子进程，并调用 execve 在子进程的上下文中执行 /cgi-bin/adder 程序。像 adder 这样的程序，被称为 CGI 程序，他们遵守 CGI 标准。在调用 execve 之前，子进程将 CGI 环境变量 QUERY_STRING 设置为 15000&213，adder 程序在运行时可以使用 getenv 函数来引用它。

3. 服务器将其他信息传递给子进程

&emsp;&emsp;CGI 定义了大量的环境变量，一个 CGI 程序在他运行时可以设置这些环境变量：

![](14.png)

4. 子进程将输出发送到哪里

&emsp;&emsp;一个 CGI 程序将它的动态内容发送到标准输出。在子进程加载并运行 CGI 程序之前，它使用 dup2 函数将标准输出重定向到和客户端相关联的已连接描述符。因此，任何 CGI 程序写到标准输出的东西都会直接到达客户端。

&emsp;&emsp;由于父进程不知道子进程生成内容的类型和大小，所以子进程要负责 Content-Type 和 Content-Length 响应报头，以及终止报头的空行。

&emsp;&emsp;对于 POST 请求，子进程也需要重定向标准输入到已连接描述符，然后，CGI 程序会从标准输入流读取请求主体中的参数内容。

## TODO 综合

&emsp;&emsp;待更新。