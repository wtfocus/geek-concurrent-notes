[toc]

## 04 | TCP 三次握手：怎么使用套接字格式建立连接？

### 服务端准备连接的过程

#### 创建套接字

-   创建函数

    -   ```C
        
        int socket(int domain, int type, int protocol)
        ```

    -   

#### bind: 设定电话号码

-   bind 函数

    -   ```C
        
        bind(int fd, sockaddr * addr, socklen_t len)
        ```

-   使用**通配地址**可以适配本机所有 ip 地址。

-   如何设置通配地址？

-   IPv4 使用 INADDR_ANY 。IPv6 使用 IN6ADDR_ANY。

    -   ```C
        
        struct sockaddr_in name;
        name.sin_addr.s_addr = htonl (INADDR_ANY); /* IPV4通配地址 */
        ```

-   初始化 IPv4 TCP 套接字的例子

    -   ```C
        
        #include <stdio.h>
        #include <stdlib.h>
        #include <sys/socket.h>
        #include <netinet/in.h>
        
        
        int make_socket (uint16_t port)
        {
          int sock;
          struct sockaddr_in name;
        
        
          /* 创建字节流类型的IPV4 socket. */
          sock = socket (PF_INET, SOCK_STREAM, 0);
          if (sock < 0)
            {
              perror ("socket");
              exit (EXIT_FAILURE);
            }
        
        
          /* 绑定到port和ip. */
          name.sin_family = AF_INET; /* IPV4 */
          name.sin_port = htons (port);  /* 指定端口 */
          name.sin_addr.s_addr = htonl (INADDR_ANY); /* 通配地址 */
          /* 把IPV4地址转换成通用地址格式，同时传递长度 */
          if (bind (sock, (struct sockaddr *) &name, sizeof (name)) < 0)
            {
              perror ("bind");
              exit (EXIT_FAILURE);
            }
        
        
          return sock
        }
        ```

#### listen：接上电话线，一切准备就绪

-   listen 函数的原型：

    -   ```C
        
        int listen (int socketfd, int backlog)
        ```



#### accept: 电话铃响起了

-   连接建立后，你可以把accept 这个函数看成是操作系统内核和应用程序之间的桥梁。它的原型是：

    -   ```C
        
        int accept(int listensockfd, struct sockaddr *cliaddr, socklen_t *addrlen)
        ```



### 客户端发起连接的过程

#### connect: 拨打电话

-   connect 函数

    -   ```C
        
        int connect(int sockfd, const struct sockaddr *servaddr, socklen_t addrlen)
        ```

#### TCP 三次握手：

-   ![img](imgs/65cef2c44480910871a0b66cac1d5529.png)

#### TCP 三次握手解读

-   具体过程：



### 总结

-   服务器端通过创建 socket，bind, listen 完成初始化，通过 accept 完成连接的建立。
-   客户端通过创建 socket，connect 发起连接建立请求。