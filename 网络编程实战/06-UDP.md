[toc]

## 06 | UDP

-   TCP VS UDP 区别？
-   为什么要使用 UDP ?

### UDP 编程

-   UDP 过程

    -   ![img](imgs/8416f0055bedce10a3c7d0416cc1f430.png)

-   recvfrom 和 sendto 是 UDP 用来接收和 发送报文的两个主要函数

    -   ```C
        
        #include <sys/socket.h>
        
        ssize_t recvfrom(int sockfd, void *buff, size_t nbytes, int flags, 
        　　　　　　　　　　struct sockaddr *from, socklen_t *addrlen); 
        
        ssize_t sendto(int sockfd, const void *buff, size_t nbytes, int flags,
                        const struct sockaddr *to, socklen_t addrlen); 
        ```

    -   

### UDP 服务端例子

-   UDP 服务器端的例子：

    -   ```c
        
        #include "lib/common.h"
        
        static int count;
        
        static void recvfrom_int(int signo) {
            printf("\nreceived %d datagrams\n", count);
            exit(0);
        }
        
        
        int main(int argc, char **argv) {
            int socket_fd;
            socket_fd = socket(AF_INET, SOCK_DGRAM, 0);
        
            struct sockaddr_in server_addr;
            bzero(&server_addr, sizeof(server_addr));
            server_addr.sin_family = AF_INET;
            server_addr.sin_addr.s_addr = htonl(INADDR_ANY);
            server_addr.sin_port = htons(SERV_PORT);
        
            bind(socket_fd, (struct sockaddr *) &server_addr, sizeof(server_addr));
        
            socklen_t client_len;
            char message[MAXLINE];
            count = 0;
        
            signal(SIGINT, recvfrom_int);
        
            struct sockaddr_in client_addr;
            client_len = sizeof(client_addr);
            for (;;) {
                int n = recvfrom(socket_fd, message, MAXLINE, 0, (struct sockaddr *) &client_addr, &client_len);
                message[n] = 0;
                printf("received %d bytes: %s\n", n, message);
        
                char send_line[MAXLINE];
                sprintf(send_line, "Hi, %s", message);
        
                sendto(socket_fd, send_line, strlen(send_line), 0, (struct sockaddr *) &client_addr, client_len);
        
                count++;
            }
        
        }
        ```

### UDP 客户端例子

-   UDP 客户端例子

    -   ```C
        
        #include "lib/common.h"
        
        # define    MAXLINE     4096
        
        int main(int argc, char **argv) {
            if (argc != 2) {
                error(1, 0, "usage: udpclient <IPaddress>");
            }
            
            int socket_fd;
            socket_fd = socket(AF_INET, SOCK_DGRAM, 0);
        
            struct sockaddr_in server_addr;
            bzero(&server_addr, sizeof(server_addr));
            server_addr.sin_family = AF_INET;
            server_addr.sin_port = htons(SERV_PORT);
            inet_pton(AF_INET, argv[1], &server_addr.sin_addr);
        
            socklen_t server_len = sizeof(server_addr);
        
            struct sockaddr *reply_addr;
            reply_addr = malloc(server_len);
        
            char send_line[MAXLINE], recv_line[MAXLINE + 1];
            socklen_t len;
            int n;
        
            while (fgets(send_line, MAXLINE, stdin) != NULL) {
                int i = strlen(send_line);
                if (send_line[i - 1] == '\n') {
                    send_line[i - 1] = 0;
                }
        
                printf("now sending %s\n", send_line);
                size_t rt = sendto(socket_fd, send_line, strlen(send_line), 0, (struct sockaddr *) &server_addr, server_len);
                if (rt < 0) {
                    error(1, errno, "send failed ");
                }
                printf("send bytes: %zu \n", rt);
        
                len = 0;
                n = recvfrom(socket_fd, recv_line, MAXLINE, 0, reply_addr, &len);
                if (n < 0)
                    error(1, errno, "recvfrom failed");
                recv_line[n] = 0;
                fputs(recv_line, stdout);
                fputs("\n", stdout);
            }
        
            exit(0);
        }
        ```

### 场景一：只运行客户端

-   ```bash
    
    $ ./udpclient 127.0.0.1
    1
    now sending g1
    send bytes: 2
    <阻塞在这里>
    ```

-   

### 场景二：先开启服务端，再开启客户端

-   ```bash
    
    $./udpserver
    received 2 bytes: g1
    received 2 bytes: g2
    ```

-   ```bash
    
    $./udpclient 127.0.0.1
    g1
    now sending g1
    send bytes: 2
    Hi, g1
    g2
    now sending g2
    send bytes: 2
    Hi, g2
    ```

### 场景三：开启服务端，再一次开启两个客户端

-   服务端

-   ```C
    
    $./udpserver
    received 2 bytes: g1
    received 2 bytes: g2
    received 2 bytes: g3
    received 2 bytes: g4
    ```

-   第一个客户端

-   ```C
    
    $./udpclient 127.0.0.1
    now sending g1
    send bytes: 2
    Hi, g1
    g3
    now sending g3
    send bytes: 2
    Hi, g3
    ```

-   第二个客户端

-   ```C
    
    $./udpclient 127.0.0.1
    now sending g2
    send bytes: 2
    Hi, g2
    g4
    now sending g4
    send bytes: 2
    Hi, g4
    ```

-   

### 总结

-   UDP 是无连接的数据报程序，和 TCP 不同，不需要三次握手建立一条连接。
-   UDP 程序时通过 recvform 和 sendto 函数直接接收发送数据报报文。