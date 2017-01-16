---
title: I/O Multiplexing & poll
date: 2016-03-18 23:46
categories: 
- Linux
tags:
- Linux
---

poll和select实现功能差不多，但poll效率比select效率高。

<!-- more -->

# 1. 什么是I/O Multiplexing

 I / O多路转接(I/O multiplexing),其基本思想是:先构造一张有关描述符的表,然后调用一个函数,它要到这些描述符中的一个已准备好进行 I / O时才返回。
在返回时,它告诉进程哪一个描述符已准备好可以进行 I / O。
IO multiplexing就是我们说的select，poll，epoll，有些地方也称这种IO方式为event driven IO。其好处就在于单个process就可以同时处理多个网络连接的IO。

![I/O 多路复用](/img/archives/io_mul.png)

poll的机制与select类似，与select在本质上没有多大差别，管理多个描述符也是进行轮询，根据描述符的状态进行处理，但是**poll没有最大文件描述符数量的限制**。poll和select同样存在一个缺点就是，包含大量文件描述符的数组被整体复制于用户态和内核的地址空间之间，而不论这些文件描述符是否就绪，它的开销随着文件描述符数量的增加而线性增大。

# 2. poll函数

```cpp
# include <poll.h>  
int poll ( struct pollfd * fds, unsigned int nfds, int timeout);
```
- timeout == INFTIM 永远等待(INFTIM 通常等于-1)
- timeout == 0 不等待
- timeout > 0 等待ttimeout毫秒

与select不同,poll不是为每个条件构造一个描述符集,而是构造一个pollfd结构数组,每个数组元素指定一个描述符编号以及对其所关心的条件。

```cpp
struct pollfd {  
int fd ;                /* file descriptor to check, or < 0 to ignore */  
short events ;          /* events of interest on fd */  
shortr events ;         /* events that occurred on fd */  
} ;  
```

简单点儿说，fd对应要监视的文件描述符，events对应需要监视的事件，revents对应实际发生的事件。

![poll-events-revents](/img/archives/poll-events-revents.png)

返回值和错误代码:
- 成功时，poll()返回结构体中revents域不为0的文件描述符个数
- 如果在超时前没有任何事件发生，poll()返回0；
- 失败时，poll()返回-1，并设置errno为下列值之一：
  + EBADF　　       一个或多个结构体中指定的文件描述符无效。
  + EFAULTfds　　 指针指向的地址超出进程的地址空间。
  + EINTR　　　　  请求的事件之前产生一个信号，调用可以重新发起。
  + EINVALnfds　　参数超出PLIMIT_NOFILE值。

# 3. 利用poll设计的web服务器

设计一个比较简单的web服务器：

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <poll.h>

#define MYPORT 8008    // the port users will be connecting to

#define BACKLOG 10     // how many pending connections queue will hold

#define BUF_SIZE 200

struct pollfd pollfds[BACKLOG + 1];
int nfds = 0;

void showclient()
{
    int i;
    printf("client count: %d\n", nfds -1);
    for (i = 0; i < BACKLOG + 1; i++) {
        printf("[%d]:%d  ", i, pollfds[i].fd);
    }
    printf("\n\n");
}

int main(void)
{
    int sock_fd, new_fd;  // listen on sock_fd, new connection on new_fd
    struct sockaddr_in server_addr;    // server address information
    struct sockaddr_in client_addr; // connector's address information
    socklen_t sin_size;
    int yes = 1;
    char buf[BUF_SIZE];
    int ret;
    int i;

    if ((sock_fd = socket(AF_INET, SOCK_STREAM, 0)) == -1) {
        perror("socket");
        exit(1);
    }

    if (setsockopt(sock_fd, SOL_SOCKET, SO_REUSEADDR, &yes, sizeof(int)) == -1) {
        perror("setsockopt");
        exit(1);
    }

    server_addr.sin_family = AF_INET;         // host byte order
    server_addr.sin_port = htons(MYPORT);     // short, network byte order
    server_addr.sin_addr.s_addr = INADDR_ANY; // automatically fill with my IP
    memset(server_addr.sin_zero, '\0', sizeof(server_addr.sin_zero));

    if (bind(sock_fd, (struct sockaddr *)&server_addr, sizeof(server_addr)) == -1) {
        perror("bind");
        exit(1);
    }

    if (listen(sock_fd, BACKLOG) == -1) {
        perror("listen");
        exit(1);
    }

    printf("listen port %d\n", MYPORT);

    sin_size = sizeof(client_addr);

    for (i = 1; i < BACKLOG + 1; i++)
    {
        pollfds[i].fd = -1;
    }


    pollfds[0].fd = sock_fd;
    pollfds[0].events = POLLIN;
    nfds++;

    while (1)
    {
        ret = poll(pollfds, nfds, -1);
        if (ret < 0)
        {
            perror("poll");
            break;
        }
        else if (ret == 0)
        {
            printf("timeout.\n");
            continue;
        }

        // add a new client
        if (pollfds[0].revents & POLLIN)
        {
            new_fd = accept(sock_fd, (struct sockaddr *)&client_addr, &sin_size);
            if (new_fd <= 0) {
                perror("accept");
                continue;
            }
            for (i = 1; i < BACKLOG + 1; i++)
            {
                if (pollfds[i].fd == -1)
                {
                    pollfds[i].fd = new_fd;
                    pollfds[i].events = POLLIN;
                    printf("add a new client: pollfds[%d] = %d \n",i,new_fd);
                    nfds++;
                    break;
                }
            }

        }

        // check every client
        for (i = 1; i < nfds; i++)
        {
            if (pollfds[i].revents & POLLIN)
            {
                ret = recv(pollfds[i].fd, buf, sizeof(buf), 0);
                char str[] = "Good,very nice!\n";

                send(pollfds[i].fd, str, sizeof(str) + 1, 0);
                if (ret <= 0) {        // client close
                    printf("client[%d] close\n", i);
                    close(pollfds[i].fd);
                    pollfds[i].fd = -1;
                    nfds--;
                } else {        // receive data
                    if (ret < BUF_SIZE)
                        memset(&buf[ret], '\0', 1);
                    printf("client[%d] send:%s\n", i, buf);
                }
            }
        }


        showclient();
    }

    // close other connections
    for (i = 0; i < nfds; i++) {
        if (pollfds[i].fd != 0) {
            close(pollfds[i].fd);
        }
    }

    exit(0);
}
```