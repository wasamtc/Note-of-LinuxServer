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

