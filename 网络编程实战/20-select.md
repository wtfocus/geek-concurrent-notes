[toc]

## 20 | select: 如何同时感知多个 I/O 事件

### 什么是 I/O 多路复用

-   I/O 多路复用的设计初衷
    -   我们可以把标准输入、套接字等都看做 I/O 的一路，多路复用的意思，就是在任何一路 I/O 有“事件”发生的情况下，通知应用程序去处理相应的 I/O 事件，这样我们的程序就变成了“多面手”，在同一时刻仿佛处理多个 I/O 事件。
-   使用 I/O 复用后，如果标准输入有数据，立即从标准输入读入数据，通过套接字发送出去; 如果套接字有数据可以读，立即可以读出数据。
-   select 函数就是这样一种常见的 I/O 多路复用技术。

### select 函数的使用方法

-   select 函数声明：

    -   ```C
        
        int select(int maxfd, fd_set *readset, fd_set *writeset, fd_set *exceptset, const struct timeval *timeout);
        
        返回：若有就绪描述符则为其数目，若超时则为0，若出错则为-1
        ```

    -   maxfd 表示的是待测试的描述基数。

    -   读描述符集合 readset

    -   写描述符集合 writeset

    -   异常描述符集合 exceptset

-   如何设置这些描述符集合呢？以下的宏可以帮助到我们

    -   ```C
        
        void FD_ZERO(fd_set *fdset);　　　　　　
        void FD_SET(int fd, fd_set *fdset);　　
        void FD_CLR(int fd, fd_set *fdset);　　　
        int  FD_ISSET(int fd, fd_set *fdset);
        ```

    -   FD_ZERO，用来将这个向量的所有元素都设置成 0。

    -   FD_SET，用来把对应套接字 fd 的元素，a[fd] 设置成 1。

    -   FD_CLR，用来把对应套接字 fd 的元素，a[fd] 设置成 0。

    -   FD_ISSET，对这个向量进行检测，判断出对应套接字元素 a[fd] 是 0 还是 1。

-   其中，0 代表不需要处理，1 代表需要处理。

-   最后一个参数是 timeval 结构体的时间：

    -   ```C
        
        struct timeval {
          long   tv_sec; /* seconds */
          long   tv_usec; /* microseconds */
        };
        ```

    -   第一个可能，是设置成空(NULL)，表示如果没有 I/O 事件发生，则 select 一直等待下去。

    -   第二个可能，是设置一个非零的值，这个表示等待固定的一段时间后，从 select 阻塞调用中返回。

    -   第三个可能，是将 tv_sec 和 tv_usec 都设置成 0，表示根本不等待，检测完毕立即返回。这种情况较少

### 程序例子

### 套接字描述符就绪条件

### 总结

