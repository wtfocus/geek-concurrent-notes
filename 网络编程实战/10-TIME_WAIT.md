[toc]

## 10 | TIME_WAIT

### TIME_WAIT 发生的场景

-   TCP 四次挥手
    -   ![img](imgs/f34823ce42a49e4eadaf642a75d14de1.png)

### TIME_WAIT 的作用

-   要从两个方面来说。
    -   首先，这样做是为了确保最后的 ACK 能让被动关闭方接收，从而帮助其正常关闭。
    -   其次，为了让旧连接的重复分节在网络中自然消失。

### TIME_WAIT 的危害

-   第一是内存资源占用，这个目前看来不是太严重，基本可以忽略。
-   第二是对端口资源的占用。

### 如何优化 TIME_WAIT ?

#### net.ipv4.tcp_max_tw_buckets

#### 调低 TCP_TIMEWAIT_LEN，重新编译系统

#### SO_LINGER 的设置

#### net.ipv4.tcp_tw_reuse：更安全的设置

-   Linux 系统对于 net.ipv4.tcp_tw_reuse 的解释

    -   ```tex
        
        Allow to reuse TIME-WAIT sockets for new connections when it is safe from protocol viewpoint. Default value is 0.It should not be changed without advice/request of technical experts.
        ```

-   那么什么是协议角度理解的安全可控呢？主要两点：

    1.  只适用于连接发起方（C/S 模型中的客户端）。
    2.  对应的 TIME_WAIT 状态的连接创建时间超过 1 秒才可以被复用。

-   使用这个选项，还有一个前提，需要打开对 TCP 时间戳的支持，即：

    -   ```bash
        net.ipv4.tcp_timestamps=1    # 默认即为 1 
        ```

    -   

### 总结

-   需要记住如下三点：
    -   TIME_WAIT 的引入是为了让 TCP 报文得以自然消失，同时为了让被动关闭方能够正常关闭。
    -   不要试图使用 SO_LINGER 设置套接字选项，跳过 TIME_WAIT。
    -   现代 Linux 系统引入更安全可控的方案，可以帮助我们尽可能地复用 TIME_WAIT 状态连接。