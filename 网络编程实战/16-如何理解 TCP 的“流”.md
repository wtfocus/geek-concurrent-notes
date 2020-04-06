[toc]

## 16 | 如何理解 TCP 的“流”？

### TCP 是一种流式协议

-   为了让大家理解 TCP 数据是流式的这个特性，我们分别从发送端和接收端来阐述。

-   如果我们考虑实际网络传输过程中的各种影响，假设发送端陆续调用 send 函数先后发送 network 和 program 报文，那么实际的发送很有可能是这个样子的。

    1.  一次性将 network 和 program 在一个 TCP 分组中发送出去。

        -   ```bash
            
            ...xxxnetworkprogramxxx...
            ```

    2.  program 部分随着 network 在一个 TCP 分组中发送出去。

        -   TCP 分组 1：

        -   ```bash
            
            ...xxxxxnetworkpro
            ```

        -   TCP 分组 2：

        -   ```bash
            
            gramxxxxxxxxxx...
            ```

    3.  network 的一部分随 TCP 分组被发送出去，另一部分和 program 一起随另一个 TCP 分组发送出去。

        -   TCP 分组 1：

        -   ```bash
            
            ...xxxxxxxxxxxnet
            ```

        -   TCP 分组 2：

        -   ```bash
            
            workprogramxxx...
            ```

-   不管是哪一种，核心问题就是：**我们不知道network 和 program 这两个报文是如何进行 TCP 分组传输的**。

-   再来看客户端，数据流的特征更明显。

-   无论发送端如何构造 TCP 分组，接收端最终收到的字节流问题像下面这样：

    -   ```bash
        
        xxxxxxxxxxxxxxxxxnetworkprogramxxxxxxxxxxxx
        ```

-   关于接收端字节流，有两点需要注意：

    1.  这里 network 和 program 的顺序肯定是会保持的。**先调用 send 函数发送的字节，总在后调用 send 函数发送字节的前面，这个是由 TCP 严格保证的**。
    2.  如果发送过程占中有 TCP 分组丢失，但是后续分组陆续到达，那么，**TCP 协议栈会缓存后续分组，直到前面丢失的分组到达**，最终，形成可以被应用程序读取的数据流。

### 网络字节排序

-   ![img](imgs/79ada2f154205f5170cf8e69bf9f59e6.png)

-   不同的系统会有两种存法：

    -   **大端字节序**
    -   **小端字节序**

-   为了保证网络字节序一致，POSIX 标准提供了如下的转换函数：

    -   ```C
        
        uint16_t htons (uint16_t hostshort)
        uint16_t ntohs (uint16_t netshort)
        uint32_t htonl (uint32_t hostlong)
        uint32_t ntohl (uint32_t netlong)
        ```

    -   这些函数可以帮助我们在主机（host）和网络（network）的格式间灵活转换。

-   如下：

    -   ```C
        
        # if __BYTE_ORDER == __BIG_ENDIAN
        /* The host byte order is the same as network byte order,
           so these functions are all just identity.  */
        # define ntohl(x) (x)
        # define ntohs(x) (x)
        # define htonl(x) (x)
        # define htons(x) (x)
        ```

    -   

### 报文读取和解析

-   只有知道了报文格式，接收端才能针对性的进行报文的读取和解析工作。
-   报文格式最重要的是如何确定报文边界。常见的报文格式有两种：
    1.  **一种是发送报把要发送的报文长度预先通过报文告知给接收端**。
    2.  **另一种是通过一些特殊字符来进行边界的划分**。

### 显式编码报文长度

#### 报文格式

-   ![img](imgs/33805892d57843a1f22830d8636e1315.png)
    -   4 个字节大小的消息长度
    -   4 个字节大小的消息类型
    -   真正要发送的数据在最后。

### 发送报文

-   发送端的程序：

    -   ```C
        int main(int argc, char **argv) {
            if (argc != 2) {
                error(1, 0, "usage: tcpclient <IPaddress>");
            }
        
            int socket_fd;
            socket_fd = socket(AF_INET, SOCK_STREAM, 0);
        
            struct sockaddr_in server_addr;
            bzero(&server_addr, sizeof(server_addr));
            server_addr.sin_family = AF_INET;
            server_addr.sin_port = htons(SERV_PORT);
            inet_pton(AF_INET, argv[1], &server_addr.sin_addr);
        
            socklen_t server_len = sizeof(server_addr);
            int connect_rt = connect(socket_fd, (struct sockaddr *) &server_addr, server_len);
            if (connect_rt < 0) {
                error(1, errno, "connect failed ");
            }
        
            struct {
                u_int32_t message_length;
                u_int32_t message_type;
                char buf[128];
            } message;
        
            int n;
        
            while (fgets(message.buf, sizeof(message.buf), stdin) != NULL) {
                n = strlen(message.buf);
                message.message_length = htonl(n);
                message.message_type = 1;
                if (send(socket_fd, (char *) &message, sizeof(message.message_length) + sizeof(message.message_type) + n, 0) <
                    0)
                    error(1, errno, "send failure");
        
            }
            exit(0);
        }
        ```

    -   

#### 解析报文：程序

-   服务器端程序对报文进行解析：

    -   ```C
        
        static int count;
        
        static void sig_int(int signo) {
            printf("\nreceived %d datagrams\n", count);
            exit(0);
        }
        
        
        int main(int argc, char **argv) {
            int listenfd;
            listenfd = socket(AF_INET, SOCK_STREAM, 0);
        
            struct sockaddr_in server_addr;
            bzero(&server_addr, sizeof(server_addr));
            server_addr.sin_family = AF_INET;
            server_addr.sin_addr.s_addr = htonl(INADDR_ANY);
            server_addr.sin_port = htons(SERV_PORT);
        
            int on = 1;
            setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on));
        
            int rt1 = bind(listenfd, (struct sockaddr *) &server_addr, sizeof(server_addr));
            if (rt1 < 0) {
                error(1, errno, "bind failed ");
            }
        
            int rt2 = listen(listenfd, LISTENQ);
            if (rt2 < 0) {
                error(1, errno, "listen failed ");
            }
        
            signal(SIGPIPE, SIG_IGN);
        
            int connfd;
            struct sockaddr_in client_addr;
            socklen_t client_len = sizeof(client_addr);
        
            if ((connfd = accept(listenfd, (struct sockaddr *) &client_addr, &client_len)) < 0) {
                error(1, errno, "bind failed ");
            }
        
            char buf[128];
            count = 0;
        
            while (1) {
                int n = read_message(connfd, buf, sizeof(buf));
                if (n < 0) {
                    error(1, errno, "error read message");
                } else if (n == 0) {
                    error(1, 0, "client closed \n");
                }
                buf[n] = 0;
                printf("received %d bytes: %s\n", n, buf);
                count++;
            }
        
            exit(0);
        
        }
        ```

    -   

#### 解析报文：readn 函数

-   这里强调下 readn 函数的语义：**读取报文预设大小的字节**。

-   readn 调用会一直循环，尝试读取预设大小的字节。

    -   ```C
        
        size_t readn(int fd, void *buffer, size_t length) {
            size_t count;
            ssize_t nread;
            char *ptr;
        
            ptr = buffer;
            count = length;
            while (count > 0) {
                nread = read(fd, ptr, count);
        
                if (nread < 0) {
                    if (errno == EINTR)
                        continue;
                    else
                        return (-1);
                } else if (nread == 0)
                    break;                /* EOF */
        
                count -= nread;
                ptr += nread;
            }
            return (length - count);        /* return >= 0 */
        }
        ```

    -   

#### 解析报文：read_message 函数

-   read_message 对报文的解析处理

    -   ```C
        
        size_t read_message(int fd, char *buffer, size_t length) {
            u_int32_t msg_length;
            u_int32_t msg_type;
            int rc;
        
            rc = readn(fd, (char *) &msg_length, sizeof(u_int32_t));
            if (rc != sizeof(u_int32_t))
                return rc < 0 ? -1 : 0;
            msg_length = ntohl(msg_length);
        
            rc = readn(fd, (char *) &msg_type, sizeof(msg_type));
            if (rc != sizeof(u_int32_t))
                return rc < 0 ? -1 : 0;
        
            if (msg_length > length) {
                return -1;
            }
        
            rc = readn(fd, buffer, msg_length);
            if (rc != msg_length)
                return rc < 0 ? -1 : 0;
            return rc;
        }
        ```

    -   

#### 实验

-   服务器端

    -   ```bash
        
        $./streamserver
        received 8 bytes: network
        received 5 bytes: good
        ```

-   客户端

    -   ```bash
        
        $./streamclient
        network
        good
        ```

    -   

### 特殊字符作为边界

-   HTTP 报文格式：

    -   ![img](imgs/6d91c7c2a0224f5d4bad32a0f488765a.png)

-   read_line 函数

    -   ```C
        
        int read_line(int fd, char *buf, int size) {
            int i = 0;
            char c = '\0';
            int n;
        
            while ((i < size - 1) && (c != '\n')) {
                n = recv(fd, &c, 1, 0);
                if (n > 0) {
                    if (c == '\r') {
                        n = recv(fd, &c, 1, MSG_PEEK);
                        if ((n > 0) && (c == '\n'))
                            recv(fd, &c, 1, 0);
                        else
                            c = '\n';
                    }
                    buf[i] = c;
                    i++;
                } else
                    c = '\n';
            }
            buf[i] = '\0';
        
            return (i);
        }
        ```

    -   

### 总结

-   TCP 数据流我决定了字节流本身是没有边界的。
-   一般我们通过显式编码报文长度、选取特殊字符区分报文边界的方式来进行报文格式的设计。
-   对报文解析的工作就是要在知道报文格式的情况下，有效地对报文信息进行还原。



