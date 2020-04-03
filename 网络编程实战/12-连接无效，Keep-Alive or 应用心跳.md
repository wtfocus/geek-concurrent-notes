[toc]

## 12 | 连接无效：使用 Keep-Alive 还是应用心跳来检测？

### 从一个例子开始

-   保持对连接有效性的检测，是我们在实战中必须要注意的一个点。

### TCP Keep-Alive 选项

-   TCP 有一个保持活跃的机制叫做 Keep-Alive。
-   这个机制的原理是这样的。
-   定义一个时间段，在这个时间段内，如果没有任何连接相关的活动，TCP 保活机制会开始作用，每隔一个时间间隔，发送一个探测报文，该探测报文包含的数据非常少，如果连续几个探测报文都没有得到响应，则认为当前的 TCP 连接已经死亡，系统内核将错误信息通知给上层应用程序。
-   上述可定义变量，分别被称为保活时间、保活时间间隔和保活探测次数。在 Linux 系统中，分别对应 sysctl 变量 net.ipv4.tcp_keepalive_time、net.ipv4.tcp_keepalive_intvl、net.ipv4.tcp_keepalive_probes，默认设置是 7200 秒（2 小时）、75 秒和 9 次探测。
-   如果开启 TCP 保活，需要考虑如下几种情况：
    1.  对端程序是正常工作的。
    2.  对端程序崩溃并重启。
    3.  对端程序崩溃，或对端由于其他原因导致报文不可达。

### 应用层探活

-   我们可以通过在应用程序中模拟 TCP Keep-Alive 机制，来完成在应用层的连接探活。
-   我们可以设计一个 PING-PONG 的机制，
-   这里有两个比较关键的点：
    1.  需要使用定时器，这可以通过使用 I/O 复用自身的机制来实现。
    2.  需要设计一个 PING-PONG 的协议。

### 消息格式设计

-   定义一个消息对象

-   ```C
    
    typedef struct {
        u_int32_t type;
        char data[1024];
    } messageObject;
    
    #define MSG_PING          1
    #define MSG_PONG          2
    #define MSG_TYPE1        11
    #define MSG_TYPE2        21
    ```

-   

### 客户端程序设计

-   客户端完全模拟 TCP Keep-Alive 的机制，在保活时间达到后，探活次数增加 1，同时向服务器端发送 PING 格式的消息，此后，以预设的保活时间间隔，不断地向服务器端发送 PING 格式的消息。如果能收到服务器端的应答，则结束保活，将保活时间置为 0。

    -   ```C
        
        #include "lib/common.h"
        #include "message_objecte.h"
        
        #define    MAXLINE     4096
        #define    KEEP_ALIVE_TIME  10
        #define    KEEP_ALIVE_INTERVAL  3
        #define    KEEP_ALIVE_PROBETIMES  3
        
        
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
        
            char recv_line[MAXLINE + 1];
            int n;
        
            fd_set readmask;
            fd_set allreads;
        
            struct timeval tv;
            int heartbeats = 0;
        
            tv.tv_sec = KEEP_ALIVE_TIME;
            tv.tv_usec = 0;
        
            messageObject messageObject;
        
            FD_ZERO(&allreads);
            FD_SET(socket_fd, &allreads);
            for (;;) {
                readmask = allreads;
                int rc = select(socket_fd + 1, &readmask, NULL, NULL, &tv);
                if (rc < 0) {
                    error(1, errno, "select failed");
                }
                if (rc == 0) {
                    if (++heartbeats > KEEP_ALIVE_PROBETIMES) {
                        error(1, 0, "connection dead\n");
                    }
                    printf("sending heartbeat #%d\n", heartbeats);
                    messageObject.type = htonl(MSG_PING);
                    rc = send(socket_fd, (char *) &messageObject, sizeof(messageObject), 0);
                    if (rc < 0) {
                        error(1, errno, "send failure");
                    }
                    tv.tv_sec = KEEP_ALIVE_INTERVAL;
                    continue;
                }
                if (FD_ISSET(socket_fd, &readmask)) {
                    n = read(socket_fd, recv_line, MAXLINE);
                    if (n < 0) {
                        error(1, errno, "read error");
                    } else if (n == 0) {
                        error(1, 0, "server terminated \n");
                    }
                    printf("received heartbeat, make heartbeats to 0 \n");
                    heartbeats = 0;
                    tv.tv_sec = KEEP_ALIVE_TIME;
                }
            }
        }
        ```

    -   

### 服务器端程序设计

-   服务器端的程序接受一个参数，这个参数设置的比较大，可以模拟连接没有响应的情况。

-   服务器端程序在接收到客户端发送来的各种消息后，进行处理，其中如果发现是 PING 类型的消息，在休眠一段时间后回复一个 PONG 消息。如果这个休眠时间很长的话，那么客户端无法快速知道服务器端是否存活。

    -   ```C
        
        #include "lib/common.h"
        #include "message_objecte.h"
        
        static int count;
        
        int main(int argc, char **argv) {
            if (argc != 2) {
                error(1, 0, "usage: tcpsever <sleepingtime>");
            }
        
            int sleepingTime = atoi(argv[1]);
        
            int listenfd;
            listenfd = socket(AF_INET, SOCK_STREAM, 0);
        
            struct sockaddr_in server_addr;
            bzero(&server_addr, sizeof(server_addr));
            server_addr.sin_family = AF_INET;
            server_addr.sin_addr.s_addr = htonl(INADDR_ANY);
            server_addr.sin_port = htons(SERV_PORT);
        
            int rt1 = bind(listenfd, (struct sockaddr *) &server_addr, sizeof(server_addr));
            if (rt1 < 0) {
                error(1, errno, "bind failed ");
            }
        
            int rt2 = listen(listenfd, LISTENQ);
            if (rt2 < 0) {
                error(1, errno, "listen failed ");
            }
        
            int connfd;
            struct sockaddr_in client_addr;
            socklen_t client_len = sizeof(client_addr);
        
            if ((connfd = accept(listenfd, (struct sockaddr *) &client_addr, &client_len)) < 0) {
                error(1, errno, "bind failed ");
            }
        
            messageObject message;
            count = 0;
        
            for (;;) {
                int n = read(connfd, (char *) &message, sizeof(messageObject));
                if (n < 0) {
                    error(1, errno, "error read");
                } else if (n == 0) {
                    error(1, 0, "client closed \n");
                }
        
                printf("received %d bytes\n", n);
                count++;
        
                switch (ntohl(message.type)) {
                    case MSG_TYPE1 :
                        printf("process  MSG_TYPE1 \n");
                        break;
        
                    case MSG_TYPE2 :
                        printf("process  MSG_TYPE2 \n");
                        break;
        
                    case MSG_PING: {
                        messageObject pong_message;
                        pong_message.type = MSG_PONG;
                        sleep(sleepingTime);
                        ssize_t rc = send(connfd, (char *) &pong_message, sizeof(pong_message), 0);
                        if (rc < 0)
                            error(1, errno, "send failure");
                        break;
                    }
        
                    default :
                        error(1, 0, "unknown message type (%d)\n", ntohl(message.type));
                }
        
            }
        
        }
        ```

    -   

### 实验

-   第一次实验，服务器端休眠时间为 60 秒。

    -   ```bash
        
        $./pingclient 127.0.0.1
        sending heartbeat #1
        sending heartbeat #2
        sending heartbeat #3
        connection dead
        ```

    -   ```bash
        
        $./pingserver 60
        received 1028 bytes
        received 1028 bytes
        ```

-   第二次实验，我们让服务器端休眠时间为 5 秒。

    -   ```bash
        
        $./pingclient 127.0.0.1
        sending heartbeat #1
        sending heartbeat #2
        received heartbeat, make heartbeats to 0
        received heartbeat, make heartbeats to 0
        sending heartbeat #1
        sending heartbeat #2
        received heartbeat, make heartbeats to 0
        received heartbeat, make heartbeats to 0
        ```

    -   ```bash
        
        $./pingserver 5
        received 1028 bytes
        received 1028 bytes
        received 1028 bytes
        received 1028 bytes
        ```

    -   

### 总结

-   虽然 TCP 没有提供系统的保活能力，让应用程序可以方便地感知连接的存活。
-   我们可以在应用程序里灵活地建立这种机制。
-   一般来说，这种机制的建立依赖于系统定时器，以及恰当的应用层报文协议。

