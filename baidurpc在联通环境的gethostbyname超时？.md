---
title: "baidurpc在联通环境的gethostbyname超时？"
author: 一张狗
lastmod: 2019-07-06 07:50:33
date: 2018-09-18 20:32:53
tags: []
---


转：

gethostname() ： 返回本地主机的标准主机名。

原型如下：
```
include <unistd.h>
int gethostname(char *name, size_t len);
```
参数说明：

这个函数需要两个参数：

接收缓冲区name，其长度必须为len字节或是更长,存获得的主机名。

接收缓冲区name的最大长度

返回值：

如果函数成功，则返回0。如果发生错误则返回-1。错误号存放在外部变量errno中。

**gethostbyname()函数说明**——用域名或主机名获取IP地址  
 包含头文件  
 #include <netdb.h>  
 #include <sys/socket.h>

函数原型  
 struct hostent *gethostbyname(const char *name);  
 这个函数的传入值是域名或者主机名，例如`www.google.cn`等等。传出值，是一个hostent的结构。如果函数调用失败，将返回NULL。

返回hostent结构体类型指针
```
struct hostent {                
    char  *h_name;            /* official name of host */                
    char **h_aliases;         /* alias list */                
    int    h_addrtype;        /* host address type */                
    int    h_length;          /* length of address */                
    char **h_addr_list;       /* list of addresses */            
}            
#define h_addr h_addr_list[0] /* for backward compatibility */
```

hostent->h_name  
 表示的是主机的规范名。例如www.google.com的规范名其实是www.l.google.com。

hostent->h_aliases  
 表示的是主机的别名.www.google.com就是google他自己的别名。有的时候，有的主机可能有好几个别名，这些，其实都是为了易于用户记忆而为自己的网站多取的名字。

hostent->h_addrtype  
 表示的是主机ip地址的类型，到底是ipv4(AF_INET)，还是pv6(AF_INET6)

hostent->h_length  
 表示的是主机ip地址的长度

hostent->h_addr_lisst  
 表示的是主机的ip地址，注意，这个是以网络字节序存储的。千万不要直接用printf带%s参数来打这个东西，会有问题的哇。所以到真正需要打印出这个IP的话，需要调用inet_ntop()。

const char *inet_ntop(int af, const void *src, char *dst, socklen_t cnt) ：  
 这个函数，是将类型为af的网络地址结构src，转换成主机序的字符串形式，存放在长度为cnt的字符串中。返回指向dst的一个指针。如果函数调用错误，返回值是NULL。

实例如下：
```
void handler(int sig){
        printf("recv a sig=%d\n",sig);
                exit(EXIT_SUCCESS);
}

#define ERR_EXIT(m) \
        do{ \
                perror(m); \
                exit(EXIT_FAILURE);\
        }while(0);
```
编译运行：
```
 gcc -o getinfo getinfo.c  
./getinfo
```
输出：
```
hostname: www.server1.com
ip: 69.172.201.208
```


http://blog.51cto.com/ty1992/1685880
https://blog.csdn.net/zouxinfox/article/details/2234225


注意：试验时主机名要是域名格式（如www.server1.com，若函数为server1时gethostbuname函数返回为NULL），gethostbyname（）函数才能获取到信息，否则返回NULL


