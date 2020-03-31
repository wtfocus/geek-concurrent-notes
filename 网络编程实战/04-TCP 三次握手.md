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

    -   