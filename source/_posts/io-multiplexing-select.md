---
title: I/O Multiplexing select
date: 2016-03-17 22:46:56
categories: 
- Linux
tags:
- Linux
---

I / O多路转接(I/O multiplexing),其基本思想是:先构造一张有关描述符的表,然后调用一个函数,它要到这些描述符中的一个已准备好进行 I / O时才返回.
在返回时,它告诉进程哪一个描述符已准备好可以进行 I / O。

<!-- more -->

# 一、什么是I/O Multiplexing？

I/O multiplexing就是我们说的select，poll，epoll，有些地方也称这种IO方式为event driven IO。其好处就在于单个process就可以同时处理多个网络连接的IO。
我们在这里仅仅来讨论select，它的基本原理就是会不断的轮询所负责的所有fdset，当某个fd有数据到达了，就通知用户进程来处理fd的读或者写事件。如果没有fd就绪，select会根据参数选择一直阻塞或者timeout。

I/O 多路复用的特点是通过一种机制一个进程能同时等待多个文件描述符，而这些文件描述符（套接字描述符）其中的任意一个进入读就绪状态，select()函数就可以返回。

![I/O 多路复用](/img/archives/io_mul.png)

# 二、select函数说明

```cpp
int select(int maxfd,fd_set *readfds,fd_set *writefds, fd_set *exceptfds,struct timeval *timeout);  
//maxfd      是需要监视的最大的文件描述符值+1
//readfds    需要检测的可读文件描述符的集合
//writefds   可写文件描述符的集合
//exceptfds 异常文件描述符的集合
```

下面的宏提供了处理这三种fd_set的方式:

> FD_CLR(inr fd,fd_set* set)；用来清除描述词组set中相关fd 的位
> FD_ISSET(int fd,fd_set *set)；用来测试描述词组set中相关fd 的位是否为真
> FD_SET（int fd,fd_set*set）；用来设置描述词组set中相关fd的位
> FD_ZERO（fd_set *set）；用来清除描述词组set的全部位

另外：

```cpp
struct timeval{
    long tv_sec;  /* seconds */
    long tv_usec; /* and microseconds */
};
```

如果参数timeout设为：

- NULL，则表示select（）没有timeout，select将一直被阻塞，直到某个文件描述符上发生了事件
- 0：仅检测描述符集合的状态，然后立即返回，并不等待外部事件的发生
- 特定的时间值：如果在指定的时间段里没有事件发生，select将超时返回

# 三、select函数返回值

执行成功则返回文件描述词状态已改变的个数，如果返回0代表在描述词状态改变前已超过timeout时间，没有返回；当有错误发生时则返回-1，错误原因存于errno，此时参数readfds，writefds，exceptfds和timeout的值变成不可预测。错误值可能为：
```
// EBADF 文件描述词为无效的或该文件已关闭
// EINTR 此调用被信号所中断
// EINVAL 参数n 为负值。
// ENOMEM 核心内存不足
```
# 四、理解Select模型：

例如,我们若编写下列代码:
```
fd_set readset, writeset;
FD_ZERO(&readset) ;
FD_ZERO(&writeset) ;
FD_SET(0, &readset);
FD_SET(3, &readset);
FD_SET(1, &writeset);
FD_SET(2, &writeset);
select (4,  &readset, &writeset, NULL, NULL);
```
那么对应的fd_set模型为：

![fdset模型](/img/archives/select.png)

# 五、如何利用select设计的web服务器：

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
  
#define MYPORT 88960    // the port users will be connecting to  
  
#define BACKLOG 10     // how many pending connections queue will hold  
  
#define BUF_SIZE 200  
  
int fd_A[BACKLOG];    // accepted connection fd  
int conn_amount;    // current connection amount  
  
void showclient()  
{  
    int i;  
    printf("client amount: %d\n", conn_amount);  
    for (i = 0; i < BACKLOG; i++) {  
        printf("[%d]:%d  ", i, fd_A[i]);  
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
  
    fd_set fdsr;  
    int maxsock;  
    struct timeval tv;  
  
    conn_amount = 0;  
    sin_size = sizeof(client_addr);  
    maxsock = sock_fd;  
    while (1) {  
        // initialize file descriptor set  
        FD_ZERO(&fdsr);  
        FD_SET(sock_fd, &fdsr);  
  
        // timeout setting  
        tv.tv_sec = 30;  
        tv.tv_usec = 0;  
  
        // add active connection to fd set  
        for (i = 0; i < BACKLOG; i++) {  
            if (fd_A[i] != 0) {  
                FD_SET(fd_A[i], &fdsr);  
            }  
        }  
  
        ret = select(maxsock + 1, &fdsr, NULL, NULL, &tv);  
        if (ret < 0) {  
            perror("select");  
            break;  
        } else if (ret == 0) {  
            printf("timeout\n");  
            continue;  
        }  
  
        // check every fd in the set  
        for (i = 0; i < conn_amount; i++) {  
            if (FD_ISSET(fd_A[i], &fdsr)) {  
                ret = recv(fd_A[i], buf, sizeof(buf), 0);  
                  
                char str[] = "Good,very nice!\n";  
                  
                send(fd_A[i],str,sizeof(str) + 1, 0);  
                  
                  
                if (ret <= 0) {        // client close  
                    printf("client[%d] close\n", i);  
                    close(fd_A[i]);  
                    FD_CLR(fd_A[i], &fdsr);  
                    fd_A[i] = 0;  
                } else {        // receive data  
                    if (ret < BUF_SIZE)  
                        memset(&buf[ret], '\0', 1);  
                    printf("client[%d] send:%s\n", i, buf);  
                }  
            }  
        }  
  
        // check whether a new connection comes  
        if (FD_ISSET(sock_fd, &fdsr)) {  
            new_fd = accept(sock_fd, (struct sockaddr *)&client_addr, &sin_size);  
            if (new_fd <= 0) {  
                perror("accept");  
                continue;  
            }  
  
            // add to fd queue  
            if (conn_amount < BACKLOG) {  
                fd_A[conn_amount++] = new_fd;  
                printf("new connection client[%d] %s:%d\n", conn_amount,  
                        inet_ntoa(client_addr.sin_addr), ntohs(client_addr.sin_port));  
                if (new_fd > maxsock)  
                    maxsock = new_fd;  
            }  
            else {  
                printf("max connections arrive, exit\n");  
                send(new_fd, "bye", 4, 0);  
                close(new_fd);  
                break;  
            }  
        }  
        showclient();  
    }  
  
    // close other connections  
    for (i = 0; i < BACKLOG; i++) {  
        if (fd_A[i] != 0) {  
            close(fd_A[i]);  
        }  
    }  
  
    exit(0);  
}  
```


