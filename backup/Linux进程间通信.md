不同进程之间一般情况下是相互独立的，也有很多方法让不同的进程间能看到同一份资源从而实现进程间通信，也叫做`IPC`
## 管道
- 管道是`Unix`中最古老的`IPC`形式
- 一个进程与另一个进程链接的数据流被称作**管道**
- 管道本质上是存在于内核中的一段缓冲区
- 管道自带同步与互斥机制
### 匿名管道
```
#include <unistd.h>

int pipe(int fd[2]);
```
- 功能:
此函数用于创建一个匿名管道
- 参数:
fd: 文件描述符数组, fd[0]代表对管道读端, fd[1]代表对管道写端
- 返回值:
成功返回`0`, 失败返回`-1`, 并设置全局变量 `errno`

匿名管道的特性是只能通过子进程继承父进程文件描述符表的方式联通进程，也就是说匿名管道只能供有亲缘关系的进程间使用
使用匿名管道
```
#include <iostream>
#include <fcntl.h>
#include <unistd.h>
#include <sys/wait.h>
#include <sys/stat.h>
#include <sys/types.h>

int main()
{
    int pipefd[2]{};
    int n = pipe(pipefd);
    if(n < 0)
    {
        perror("pipe create error");
        return 0;
    }
    pid_t id = fork();
    if(id < 0)
    {
        perror("fork error");
        return 0;
    }
    else if(id == 0)
    {
        close(pipefd[1]);
        char buf[1024]{};
        ssize_t ret = read(pipefd[0], buf, sizeof(buf));
        if(ret < 0)
        {
            perror("read error");
            exit(1);
        }
        std::cout << buf << std::endl;
        exit(1);
    }
    close(pipefd[0]);
    char buf[1024]{"i am father"};
    ssize_t ret = write(pipefd[1], buf, sizeof(buf));
    if(ret < 0)
    {
        perror("write error");
        return 0;
    }
    waitpid(id, nullptr, 0);
}
```
> 使用管道进行进程间通信的方式即为对`fd`的文件读写
### 管道特点
- 管道已满时，
    - 阻塞模式  `write`接口阻塞
    - 非阻塞模式 `write`返回`-1`，`errno`值被设为`EAGAIN`
- 管道为空时，
    - 阻塞模式  `read`端口阻塞
    - 非阻塞模式 `read`返回`-1`，`errno`值被设为`EAGAIN`
- 写端关闭，`read`不断返回`0`
- 读端关闭，写端进程被`SIGPIPE`信号杀死
- 写入数据量不大于`PIPE_BUF`时，Linux保证写入操作原子性，反之则不保证
- 管道生命周期随进程
- 管道是半双⼯的，需要进程双方通信时，需要建⽴起两个管道
### 命名管道
## System V
### System V 共享内存
### System V 消息队列
### System V 信号量
