@[toc]
# 进程
>操作系统资源分配的基本单位，是一个独立的运行环境。每个进程都有自己独立的内存空间、文件描述符等资源。每个进程拥有独立的地址空间，包括代码段、数据段、堆栈等；而线程共享相同的地址空间。
# 创建子进程
```c
#include <sys/types.h>
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <errno.h>
#include <wait.h>

int main(int argc, char const *argv[])
{
    pid_t pid = -1;
    /**
     * @brief pid_t fork(void) 复制进程，创建当前进程的子进程
     * @return 创建失败返回-1
     *         创建成功，返回0表示子进程，返回值大于0表示父进程，且返回值为子进程 ID
    */
    pid = fork();
    if (0 > pid)
    {   /* ======== 进程创建失败 ======== */
        perror("fork");
        exit(EXIT_FAILURE);
    } else if (0 == pid)
    {   /* ======== 子进程部分 ======== */
        printf("这是子进程 ID 为 [%d].\n", getpid());      // 当前进程（子进程） ID
    } else 
    {   /* ======== 父进程部分 ======== */
	    printf("这是父进程 ID 为 [%d].\n", getpid());      // 当前进程（父进程） ID
        waitpid(pid, NULL, 0);                   		  // 等待回收子进程
        												  // 若父进程先于子进程完成运行
        												  // 子进程将会变为孤儿进程
    }
    return 0;
}
```
> 程序运行结果：通常子进程 ID 是 父进程 ID + 1。
> ![创建子进程](https://i-blog.csdnimg.cn/direct/919a2a60f17a48919293766647cd359f.png)

# 进程间通信（IPC）
> 进程间通信是指不同进程之间的数据交换
## 管道
- 匿名管道：用于具有亲缘关系（例如父子进程）的进程之间的通信。数据在管道的两端流动，一个进程写入数据，另一个进程从管道读取数据。
```c
#include <sys/types.h>
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <errno.h>
#include <wait.h>
#include <string.h>

int main(int argc, char const *argv[])
{
    pid_t pid = -1;
    // 管道描述符包含两个部分，读管道和写管道
    int pipefd[2];
    char rmsg;
    char massage[100] = "Unnamed pipe test!"; 
    /**
     * @brief int pipe (int __pipedes[2]) 创建管道
     * @return 创建失败返回-1
     *         创建成功，返回管道描述符数组，第一个元素表示读，第二个元素表示写
    */
    if (-1 == pipe(pipefd))
    {   /* ======== 管道创建失败 ======== */
        perror("pipe");
        exit(EXIT_FAILURE);
    }

    pid = fork();
    if (-1 == pid)
    {   /* ======== 进程创建失败 ======== */
        perror("fork");
        exit(EXIT_FAILURE);
    } else if (0 == pid)
    {   /* ======== 子进程读取数据 ======== */
        close(pipefd[1]);                                // 关闭管道写端
        printf("子进程 [%d] 读取数据.\n", getpid());
        printf("读取内容为：");
        // 逐字符读取
        while (read(pipefd[0], &rmsg, 1) > 0)
        {
            printf("%c", rmsg);
        }
        printf("\n");
        close(pipefd[0]);                                // 关闭管道读端
        exit(EXIT_SUCCESS);
    } else {/* ======== 父进程写入数据 ======== */
        close(pipefd[0]);                                // 关闭管道读端
	    printf("父进程 [%d] 写入数据.\n", getpid());
        write(pipefd[1], massage, strlen(massage));
        close(pipefd[1]);                                // 关闭管道写端
        waitpid(pid, NULL, 0);                           // 等待回收子进程
    }
    return 0;
}
```
> 程序运行结果如下：管道描述符的读写操作默认为阻塞式，可通过 fcntl() 修改。![匿名管道](https://i-blog.csdnimg.cn/direct/9c605a3bae08410398bfedab37984fea.png)


- 命名管道：可以在不具有亲缘关系的进程之间进行通信。它们在文件系统中有一个名字，允许不同进程通过名字来访问。
>进程1：创建写端 fifo_write.c
```c
#include <sys/types.h>
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <errno.h>
#include <wait.h>
#include <sys/stat.h>
#include <string.h>
#include <fcntl.h>

int main(int argc, char const *argv[])
{
    /* ======== 创建变量 ======== */
    char *pipe_path = "./Myfifo";           // 有名管道文件路径
    int fd = -1;                            // 管道文件描述符
    char buf[100];                          // 数据缓冲数组

    /**
     * @brief int mkfifo (const char *__path, __mode_t __mode) 创建管道
     * @return 创建失败返回-1
     *         创建成功，返回0
    */
    if (0 != mkfifo(pipe_path, 0664))
    {   /* ======== 管道创建失败 ======== */
        perror("mkfifo");
        exit(EXIT_FAILURE);
    }

    fd = open(pipe_path, O_WRONLY);
    if (-1 == fd)
    {   /* ======== 文件打开失败 ======== */
        perror("open");
        exit(EXIT_FAILURE);
    }

    /* ======== 将终端输入的内容写入管道 ======== */
    while(read(STDIN_FILENO, buf, 100) > 0)
    {  
        write(fd, buf, strlen(buf));
        memset(buf, 0, 100);
    }

    close(fd);                                  // 关闭管道文件描述符
    unlink(pipe_path);                          // 释放管道
    return 0;
}
```
>进程2：创建读端 fifo_read.c
```c
#include <sys/types.h>
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <errno.h>
#include <wait.h>
#include <sys/stat.h>
#include <string.h>
#include <fcntl.h>

int main(int argc, char const *argv[])
{
    /* ======== 创建变量 ======== */
    char *pipe_path = "./Myfifo";           // 有名管道文件路径
    int fd = -1;                            // 管道文件描述符
    char buf[100];                          // 数据缓冲数组

    fd = open(pipe_path, O_RDONLY);
    if (-1 == fd)
    {   /* ======== 文件打开失败 ======== */
        perror("open");
        exit(EXIT_FAILURE);
    }
    
    /* ======== 读取管道中的数据输出至终端 ======== */
    while(read(fd, buf, strlen(buf)) > 0)
    {
        printf("%s", buf);
    }

    close(fd);                          // 关闭管道文件描述符
    return 0;
}
```
> note：通信完成需要是使用 unlink 释放管道，不建议重复使用管道。
> 由于需要实现不同进程间通信，所以打开两个终端，分别执行读、写，程序运行结果如下：
> ![终端1写入](https://i-blog.csdnimg.cn/direct/6725b136f8464924a47c190dbe826432.png)
![终端2读取](https://i-blog.csdnimg.cn/direct/efdfd321eb8f4eedbed8ec9b082fbc1a.png)
> 若要结束进程，需要按Ctrl + D

## 共享内存
允许多个进程访问同一块内存区域，高效交换数据。进程通过**内存映射**一块共享内存区域来实现通信，但共享内存本身并不提供同步机制，因此需要使用其他同步机制来防止出现竞态条件。（可以自定义空间大小）

```c
#include <stdio.h>
#include <errno.h>
#include <wait.h>
#include <sys/types.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <sys/mman.h>
#include <string.h>

int main(int argc, const char *argv[])
{
    /* ======== 创建变量 ======== */
    int share_size = 1024;                      // 共享内存空间大小
    char *share;                                // 共享内存起始地址指针
    char *share_name = "/shn_letter";           // 共享内存对象名称

    /**
     * int shm_open (const char *__name, int __oflag, mode_t __mode)
     * 创建共享内存对象
    */
    int shm_fd = shm_open(share_name, O_RDWR | O_CREAT, 0644);
    if (-1 == shm_fd)
    {   /* ======== 共享内存对象创建失败 ======== */
        perror("shm_open");
        exit(EXIT_FAILURE);
    }

    /**
     * int ftruncate(int fd, off_t length);
     * 设置共享内存大小
    */
    int ret = ftruncate(shm_fd, share_size);
    if (-1 == ret)
    {   /* ======== 共享内存大小设置失败 ======== */
        perror("ftruncate");
        exit(EXIT_FAILURE);
    }

    /**
     * void *mmap (void *__addr, size_t __len, int __prot, int __flags, int __fd, __off_t __offset)
     * 内存映射
    */
    share = mmap(NULL, share_size, PROT_WRITE | PROT_READ, MAP_SHARED, shm_fd, 0);
    if (share == MAP_FAILED)
    {   /* ======== 内存映射失败 ======== */
        perror("mmap");
        exit(EXIT_FAILURE);
    }
    close(shm_fd);                               // 映射完成，关闭共享对象描述符

    pid_t pid = fork();                          // 创建子进程
    if (-1 == pid)
    {   /* ======== 进程创建失败 ======== */
        perror("fork");
        exit(EXIT_FAILURE);
    }

    if (0 == pid)
    {   /* ======== 子进程 ======== */
        printf("子进程 [%d] 发送数据成功！\n", getpid());
        strcpy(share, "hello world!");
    } else
    {   /* ======== 父进程 ======== */
        sleep(1);									// 手动休眠 1s 模拟进程同步
        printf("父进程 [%d] 接收到子进程 [%d] 数据：[%s].\n", getpid(), pid, share);
        waitpid(pid, NULL, WUNTRACED);              // 回收子进程
    }

    /**
     * int munmap (void *__addr, size_t __len) __THROW;
     * 释放映射区
    */
    int mun_share = munmap(share, share_size);
    if (-1 == mun_share)
    {   /* ======== 映射区释放失败 ======== */
        perror("munmap");
        exit(EXIT_FAILURE);
    }

    shm_unlink(share_name);                         // 释放共享内存对象
    return 0;
}
```
>程序运行结果如下：
![共享内存](https://i-blog.csdnimg.cn/direct/86b2f79f4ca0439c827b439d41cdfa23.png)
使用后须要释放共享内存空间和共享内存对象，shm_open() 创建共享内存对象，传输信息并不存储在其中，而是在内存中，所以读写时直接操作内存，不需要像其他通信方式一样调用读、写函数。

## 消息队列
将消息存储在队列中，采用 FIFO 方式存取数据
- 当**队列满**时发送函数`mq_timedsed`允许设置超时等待，若在等待时间内有进程从队列中接收数据则可继续发送
- 当**队列空**时接收函数`mq_timedreceive`允许设置超时等待，若在等待时间内有进程向队列中写入数据则可继续读取
```c
#include <sys/types.h>
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <errno.h>
#include <wait.h>
#include <string.h>
#include <mqueue.h>
#include <sys/stat.h>
#include <time.h>

int main(int argc, char const *argv[])
{
    /* ======== 创建变量 ======== */
    char *mq_name = "/MyQueue";     // 消息队列对象
    // sprintf(mq_name, "/queue%d", getpid());
    struct mq_attr attr;            // 消息队列属性结构体
    struct timespec spec;
    attr.mq_curmsgs = 0;            // 当前消息数量（不重要）
    attr.mq_flags = 0;              // 消息队列标志（不重要）
    attr.mq_maxmsg = 10;            // 消息队列最大长度
    attr.mq_msgsize = 100;          // 消息最大字节数
    char receive_buf[100];          // 接收缓存数组
    char send_buf[100];             // 发送缓存数组

    /**
     * mqd_t mq_open(const char *name, int oflag, mode_t mode, struct mq_attr *attr)
     * 创建消息队列
     * @return 失败返回-1
     *         成功返回消息队列描述符
    */
    mqd_t mq_fd = mq_open(mq_name, O_RDWR | O_CREAT, 0644, &attr);
    if ((mqd_t)-1 == mq_fd)
    {   /* ======== 消息队列创建失败 ======== */
        perror("mq_open");
        exit(EXIT_FAILURE);
    }

    pid_t pid = fork();
    if ((pid_t)-1 == pid)
    {   /* ======== 创建进程失败 ======== */
        perror("fork");
        exit(EXIT_FAILURE);
    }

    if (0 == pid)
    {   /* ======== 子进程 ======== */
        /**
         * ssize_t mq_timedreceive(mqd_t mqdes, char *msg_ptr,
                          size_t msg_len, unsigned int *msg_prio,
                          const struct timespec *abs_timeout);
           接收消息  
        */
        for (int i = 0; i < 10; i++)
        {
            memset(receive_buf, 0, 100);
            clock_gettime(0, &spec);        // 获取当前时间
            spec.tv_sec += 2;               // 超时等待时间 2s
            if (-1 == mq_timedreceive(mq_fd, receive_buf, 100, NULL, &spec))
            {
                perror("mq_timedreceive");
                // exit(EXIT_FAILURE);
            }
            printf("子进程接收数据：%s\n", receive_buf);
        }
    } else
    {   /* ======== 父进程 ======== */
        /**
         * int mq_timedsend(mqd_t mqdes, const char *msg_ptr,
                     size_t msg_len, unsigned int msg_prio,
                     const struct timespec *abs_timeout);
         * 发送消息
        */
        for (int i = 0; i < 10; i++)
        {
            memset(send_buf, 0, 100);
            sprintf(send_buf, "父进程第 [%d] 次发送数据。", i+1);
            clock_gettime(0, &spec);            // 获取当前时间
            spec.tv_sec += 2;                   // 超时等待时间 2s
            if (-1 == mq_timedsend(mq_fd, send_buf, strlen(send_buf), 0, &spec))
            {
                perror("mq_timedsend");
                exit(EXIT_FAILURE);
            }
            sleep(1);
        }
        waitpid(pid, NULL, WUNTRACED);            // 回收子进程
        mq_unlink(mq_name);                       // 释放消息队列
    }
    mq_close(mq_fd);                              // 关闭消息队列描述符
    return 0;
}
```
> 程序运行结果如下：![消息队列](https://i-blog.csdnimg.cn/direct/765debdf0cd84bdcac423a6f7bccbb79.png)
# 孤儿进程和僵尸进程
**孤儿进程**：父进程先于其子进程结束，则子进程成为孤儿进程，子进程资源由 init 进程（PID＝1）回收

**僵尸进程**：子进程结束，但父进程没有使用 `wait` 或者 `waitpid` 回收子进程，子进程描述符仍然存储在系统中
>如果父进程在子进程 `exit` 之后，没有及时回收，出现僵尸进程，可以用 `ps` 命令去查看，进程状态为“Z”
# 守护进程
独立于终端，在后台运行，大多数服务器的实现方式就是基于守护进程
- 保护服务端稳定，脱离终端，后台稳定运行
- 通过修改已有进程参数使其变为守护进程
	- `fork` 子进程，父进程 `exit`，使子进程成为**孤儿进程**，被挂载到系统进程
	- 子进程调用 `setsid` 创建一个新的会话组，并将其 ID 作为新会话组的 ID，使进程摆脱与当前控制台联系
	- 更改工作目录至根目录 `chdir`
	- 使用 `umask(0)` 设置文件权限掩码（设置允许当前进程创建文件或者目录可操作权限为`rwxrwxrwx`）
	- 关闭不需要的文件描述符
	- 处理信号，只接收 `kill` 信号
> 系统函数 `daemon` 可以快速创建守护进程

# 进程同步
在使用共享内存时，多个进程同时访问同一资源，若不进行同步，则可能输出结果错误。
>l例如：父子进程分别对共享内存进行 1000 次累加，预期输出结果为 2000，而实际输出结果与预期不符。

```c
#include <stdio.h>
#include <errno.h>
#include <wait.h>
#include <sys/types.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <sys/mman.h>
#include <string.h>

int main(int argc, const char *argv[])
{
    const int count = 1000;						// 累加次数
    /* ======== 创建变量 ======== */
    int share_size = 1024;                      // 共享内存空间大小
    int *share;                                 // 共享内存起始地址指针
    char *share_name = "/shn_letter";           // 共享内存对象名称

    int shm_fd = shm_open(share_name, O_RDWR | O_CREAT, 0644);
    if (-1 == shm_fd)
    {
        perror("shm_open");
        exit(EXIT_FAILURE);
    }
    int ret = ftruncate(shm_fd, share_size);
    if (-1 == ret)
    {
        perror("ftruncate");
        exit(EXIT_FAILURE);
    }
    share = mmap(NULL, share_size, PROT_WRITE | PROT_READ, MAP_SHARED, shm_fd, 0);
    if (share == MAP_FAILED)
    {
        perror("mmap");
        exit(EXIT_FAILURE);
    }
    close(shm_fd);                               // 映射完成，关闭共享对象描述符
    *share = 0;

    int pid = fork();
    if (pid < 0) 
    {
        perror("fork");
    }
    else if (pid == 0)
    {
        // 子进程
        for (int i = 0; i < count; i++)
        {
            (*share)++;
            printf("子进程执行，count为：%d\n", (*share));
        }
        exit(EXIT_SUCCESS);
    }
    else if (pid > 0)
    {
        // 父进程
        for (int j = 0; j < count; j++)
        {
            (*share)++;
            printf("子进程执行，count为：%d\n", (*share));
        }
        exit(EXIT_SUCCESS);
    }

    waitpid(pid, NULL, 0);
    shm_unlink(share_name);
    return 0;
}
```
![共享内存竞态条件](https://i-blog.csdnimg.cn/direct/c1b9e3f1a5054c78a5044c44a16f6ff7.png)
## 信号量
更新中... ...

> 欢迎批评指正！！！

