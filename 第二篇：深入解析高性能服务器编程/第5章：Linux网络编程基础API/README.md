# Linux网络基础API

本章主要介绍一些API，包括：socket地址API，socket基础API，网络信息API。

## 一.socket地址API

### 1.主机字节序和网络字节序

首先我们要搞清楚大端字节序和小端字节序，大端字节序是一个整数的高位字节对应内存的低地址处，小端字节序是一个整数的高位字节对应内存的高地址处，注意一个整数的高位字节是指左边的那些位。可用下面的程序测试。

```C
#include <stdio.h>

int main()
{
    union test{
        short value;
        char union_bytes[sizeof(short)];
    };

    test temp;
    temp.value = 0x0102;
    if(temp.union_bytes[0] == 1 && temp.union_bytes[1] == 2){
        printf("big endian\n");
    } else if(temp.union_bytes[1] == 1 && temp.union_bytes[0] == 2){
        printf("little endian\n");
    } else{
        printf("unknown\n");
    }
}
```

由于有的主机采用的是大端字节序（如Java虚拟机），有的采用小端字节序，所以在网络中传输数据时若不进行特定的转换则可能会导致错误的数据解释，所以我们在将数据传输到网络前，一般先转换为网络字节序（大端字节序），接收端根据自身情况进行转换。Linux在<netinet/in.h>中提供了四个转换函数。htonl, htons, ntohl, ntohs。

### 2.通用socket地址

地址族对应协议族，从而使得sa_data有不同的解释，如果为AF_INET，则sa_data为16bit的端口号和32bit的IP地址。

### 3.专用socket地址

Linux为每个协议族提供了专用的socket地址结构体，不过要主要的是这些专用socket地址在实际使用的时候（例如作为函数参数）都要转换为通用socket地址（因为所有socket编程接口使用的地址参数都是通用socket），强制转换即可。

### 4.IP地址转换函数

IP地址转换函数的主要作用为把字符串形式的IP地址转换为网络字节序，把网络字节序的IP地址转换为字符串形式。

### 5.创建socket

指定三个参数，协议族，传输协议，具体协议。

### 6.命名socket

所谓命名socket，是指服务器的socket要与某个地址绑定。

### 7.监听socket

服务器socket命名好之后，可以开始在这个socket上监听并设置最大队列。监听队列也就是等待accept的队列。

### 8.接收连接

accept接收监听队列中的一个连接，无论队列里面的连接是断网了还是退出了。

accept返回一个新的socket及绑定的地址。

### 9.发起连接

对于客户端，常常需要通过connect发起连接。

### 10.关闭连接

要注意关闭连接，不然可能一个端口长期被某个连接占据，close是把fd上的引用减1,（fork的时候引用数会加1），还可以使用功能更加强大的shutdown。

下面附上简单的客户端以及服务器程序

```C++
#include "includes/util.h"
#include <sys/socket.h>
#include <arpa/inet.h>
#include <string.h>
#include <unistd.h>
#include <stdlib.h>

int main(int argc, char* argv[])
{
    if(argc != 3){
        pmess(1, "参数数量不符合要求\n");
        return 0;
    }

    const char* ip = argv[1];
    int port = atoi(argv[2]);


    int sockfd = socket(PF_INET, SOCK_STREAM, 0);

    sockaddr_in serv_addr;
    bzero(&serv_addr, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_addr.s_addr = inet_addr(ip);
    serv_addr.sin_port = htons(port);

    int ret = connect(sockfd, (sockaddr*)&serv_addr, sizeof(serv_addr));
    pmess(ret == -1, "socket connnet error\n");

    sleep(20);
    pmess(1, "客户端断开连接\n");
}
```

```C
#include <stdio.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <util.h>
#include <string.h>
#include <stdlib.h>
#include <signal.h>
#include <unistd.h>

static int temp;

static void handle_term(int sig){
    pmess(1, "SIGINT触发，服务器关闭\n");
    close(temp);
    exit(0);
}

int main(int argc, char* argv[])
{
    signal(SIGINT, handle_term);

    if(argc != 3){
        pmess(1, "参数数量不符合要求\n");
        return 0;
    }

    const char* ip = argv[1];
    int port = atoi(argv[2]);

    sockaddr_in serv_addr;
    bzero(&serv_addr, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(port);
    serv_addr.sin_addr.s_addr = inet_addr(ip);

    int welcome_fd = socket(PF_INET, SOCK_STREAM, 0);
    pmess(welcome_fd == -1, "socket create error!\n");
    temp = welcome_fd;

    int ret = bind(welcome_fd, (sockaddr*)(&serv_addr), sizeof(serv_addr));
    pmess(ret == -1, "socket bind  error\n");

    ret = listen(welcome_fd, 5);
    pmess(ret == -1, "socket listen error\n");

    sockaddr_in clnt_addr;
    bzero(&clnt_addr, sizeof(clnt_addr));
    socklen_t clnt_addr_len = sizeof(clnt_addr);
    while(true){
        int conn_fd = accept(welcome_fd, (sockaddr*)&clnt_addr, &clnt_addr_len);
        
        if(conn_fd < 0){
            pmess(1, "连接错误\n");
        } else{
            printf("client ip : %s, port : %d, fd : %d\n", (inet_ntoa(clnt_addr.sin_addr)), ntohs(clnt_addr.sin_port), conn_fd);
        }

    }
    pmess(1, "服务器已停止\n");
    close(welcome_fd);
    return 0;
}
```

下面是输出程序

```C++
#ifndef UTIL_H
#define UTIL_H
#include <stdio.h>

void pmess(bool flag, const char* message){
    if(flag == 1){
        printf("%s\n", message);
    }
}

#endif
```

## 二.数据读写

### 1.TCP数据读写

搞了好久，主要各种各样的错误，还有这个WSL1好多都不行（传给紧急数据都不行，恼火），升级到WSL2后好多了，还有就是要根据自身情况来设置条件，跟书上一样有时候是不行的。

### 2.UDP数据读写

跟TCP最不同的是UDP每次读要存储对方地址，每次写要加上对方地址。

### 3.通用数据读写函数

Linux还提供了两个UDP和TCP都能使用的数据读写函数，不过看起来就很麻烦。

下面是两个读写程序，至于util.h上面已经给出。

```C++
#include "includes/util.h"
#include <sys/socket.h>
#include <arpa/inet.h>
#include <string.h>
#include <unistd.h>
#include <stdlib.h>

int main(int argc, char* argv[])
{
    if(argc != 3){
        pmess(1, "参数数量不符合要求\n");
        return 0;
    }

    const char* ip = argv[1];
    int port = atoi(argv[2]);


    int sockfd = socket(PF_INET, SOCK_STREAM, 0);

    sockaddr_in serv_addr;
    bzero(&serv_addr, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_addr.s_addr = inet_addr(ip);
    serv_addr.sin_port = htons(port);

    int ret = connect(sockfd, (sockaddr*)&serv_addr, sizeof(serv_addr));
    pmess(ret == -1, "socket connnet error\n");

    const char* normal_data = "123";
    const char* urgent_data = "abc";

    send(sockfd, normal_data, strlen(normal_data), 0);
    send(sockfd, urgent_data, strlen(urgent_data), MSG_OOB);
    send(sockfd, normal_data, strlen(normal_data), 0);

    sleep(20);
    close(sockfd);
    pmess(1, "客户端断开连接\n");
}
```

```C++
#include <stdio.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <util.h>
#include <string.h>
#include <stdlib.h>
#include <signal.h>
#include <unistd.h>

#define BUF_SIZE 1024



static int temp;

static void handle_term(int sig){
    pmess(1, "SIGINT触发, 服务器关闭\n");
    close(temp);
    exit(0);
}

int main(int argc, char* argv[])
{
    signal(SIGINT, handle_term);

    if(argc != 3){
        pmess(1, "参数数量不符合要求\n");
        return 0;
    }

    const char* ip = argv[1];
    int port = atoi(argv[2]);

    sockaddr_in serv_addr;
    bzero(&serv_addr, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(port);
    serv_addr.sin_addr.s_addr = inet_addr(ip);

    int welcome_fd = socket(PF_INET, SOCK_STREAM, 0);
    pmess(welcome_fd == -1, "socket create error!\n");
    temp = welcome_fd;

    int ret = bind(welcome_fd, (sockaddr*)(&serv_addr), sizeof(serv_addr));
    pmess(ret == -1, "socket bind  error\n");

    ret = listen(welcome_fd, 5);
    pmess(ret == -1, "socket listen error\n");

    sockaddr_in clnt_addr;
    bzero(&clnt_addr, sizeof(clnt_addr));
    socklen_t clnt_addr_len = sizeof(clnt_addr);
    while(true){

        char buf[BUF_SIZE];
        memset(buf, '\0', BUF_SIZE);

        int conn_fd = accept(welcome_fd, (sockaddr*)&clnt_addr, &clnt_addr_len);
        
        if(conn_fd < 0){
            pmess(1, "连接错误\n");
        } else{
            printf("client ip : %s, port : %d, fd : %d\n", (inet_ntoa(clnt_addr.sin_addr)), ntohs(clnt_addr.sin_port), conn_fd);
        }

        memset(buf, '\0', BUF_SIZE);
        ret = recv(conn_fd, buf, BUF_SIZE-1, 0);
        printf("字节数：%d, 内容：%s\n", ret, buf);
        memset(buf, '\0', BUF_SIZE);
        ret = recv(conn_fd, buf, BUF_SIZE-1, MSG_OOB);
        printf("字节数：%d, 内容：%s\n", ret, buf);
        memset(buf, '\0', BUF_SIZE);
        ret = recv(conn_fd, buf, BUF_SIZE-1, 0);
        printf("字节数：%d, 内容：%s\n", ret, buf);


    }
    pmess(1, "服务器已停止\n");
    close(welcome_fd);
    return 0;
}
```

## 三.带外标记

刚刚我们写的程序接收紧急数据是知道要发送紧急数据了，但是正常使用中我们总不能未卜先知吧，所以使用的还是一个函数来判断下一个数据是否是带外数据。这个我遇到一个很诡异的问题，就是接受紧急数据后sockatmark还是一直返回1，

## 四.socket选项

socket本身是可以设置很多选项的，这些选项是对通信的一些设置，例如我们接受带外数据时可以利用设置选项使得带外数据存放在普通缓存区。

## 五.网络信息API

也有很多通过主机名或服务名以及其他信息来获得主机信息或其他信息的网络API。
