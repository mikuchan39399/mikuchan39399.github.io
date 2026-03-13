不同进程之间一般情况下是相互独立的，也有很多方法让不同的进程间能看到同一份资源从而实现进程间通信，也叫做`IPC`
## 管道
- 管道是`Unix`中最古老的`IPC`形式
- 一个进程与另一个进程链接的数据流被称作**管道**
- 管道本质上是存在于内核中的一段缓冲区
- 管道自带同步与互斥机制
- 管道生命周期随进程
### 匿名管道
```
#include <unistd.h>

int pipe(int fd[2]);
```
- 功能:
此函数用于创建一个匿名管道`pipe`，并打开对其的读写端口
- 参数:
fd: 文件描述符数组, fd[0]代表对管道读端, fd[1]代表对管道写端
- 返回值:
成功返回`0`, 失败返回`-1`, 并设置全局变量 `errno`

匿名管道的特性是只能通过子进程继承父进程文件描述符表的方式联通进程，也就是说匿名管道只能供有亲缘关系的进程间使用
使用匿名管道
```cpp
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
命名管道`FIFO`是一种特殊的，不刷新至磁盘的文件，相较于匿名管道，命名管道可用于任意两个进程间的通信
`命令行创建命名管道`
```
mkfifo filename
```
`程序调用函数创建`
```c
#include <sys/types.h>
#include <sys/stat.h>

int mkfifo (const char* filename, mode_t mode);
```
命名管道在创建出来之后需要主动`open`，这是与匿名管道之间的区别所在，创建打开完成之后，两者之间具有相同的语义
- 读方式打开命名管道
   - 阻塞模式：阻塞直到有其他进程以写方式打开该命名管道
   - 非阻塞模式：立刻返回对应文件描述符
- 写方式打开命名管道
   - 阻塞模式：阻塞直到有相应进程为读⽽打开该命名管道
   - 非阻塞模式：立刻返回`-1`, `errno`被设置为`ENXIO`
```
#pragma once
#include <iostream>
#include <string>
#include <sys/stat.h>
#include <sys/types.h>
#include <unistd.h>
#include <fcntl.h>

constexpr int defualt_fd = -1;
constexpr std::size_t SIZE = 4096;

class NamedPipe
{
public:
    NamedPipe(const std::string name)
    : _name(name)
    , _fd(defualt_fd)
    {}

    ~NamedPipe() 
    {
        Close();
        Remove();
    }

    bool Create(mode_t mode)
    {
        int n = mkfifo(_name.c_str(), mode);
        if(n == 0)
        {
            std::cout << "mkfifo create success" << std::endl;
            return true;
        }
        else
        {
            perror("mkfifo");
            return false;
        }
    }

    bool Close()
    {
        if(_fd == defualt_fd) 
            return false;
        else 
        {
            close(_fd);
            return true;
        }
    }

    bool OpenForRead()
    {
        _fd = open(_name.c_str(), O_RDONLY);
        if(_fd < 0)
        {
            perror("open");
            return false;
        }
        std::cout << "open file success" << std::endl;
        return true;
    }

    bool OpenForWrite()
    {
        _fd = open(_name.c_str(), O_WRONLY);
        if(_fd < 0)
        {
            perror("open");
            return false;
        }
        std::cout << "open file success" << std::endl;
        return true;
    }

    bool Read(std::string& out)
    {
        char buffer[SIZE]{};
        ssize_t num = 0;
        do 
        {
            num = read(_fd, buffer, sizeof(buffer)); 
        } while (num < 0 && errno == EINTR);

        if(num > 0)
        {
            out.assign(buffer, num);
            return true;
        }
        return false;
    }

    void Write(const std::string& in)
    {
        write(_fd, in.c_str(), in.size());
    }

    void Remove()
    {
        int m = unlink(_name.c_str());
        (void)m;
    }
private:
    std::string _name;
    int _fd;
};
```
## System V
### System V 共享内存
- 共享内存是`IPC`速度最快的方式，直接往内核态中写入数据，它没有任何同步或互斥机制
- 在实际工程中，共享内存几乎必须配合其他同步机制一起使用
- `System V IPC` 生命周期随内核
```c
#include <sys/ipc.h>
#include <sys/shm.h>
#include <sys/types.h>
key_t ftok(const char *pathname, int proj_id)
```
- 功能：将一个已存在的路径名和一个项目 ID 转换成一个 System V IPC 键值
- 键值供内核辨别资源的唯一性
- 参数
    -  `pathname` 已存在的文件路径名
    - `proj_id`项目 ID
- 返回值：键值


```c
#include <sys/ipc.h>
#include <sys/shm.h>
#include <sys/types.h>
int shmget(key_t key, size_t size, int shmflg)
```
- 功能：根据给定的键值，创建一块新的共享内存，或者获取一块已经存在的共享内存，返回共享内存的标识符
- 键值供内核辨别资源的唯一性
- 参数：
    -  `key` 对应键值
    - `size` 若键值无对应共享内存，创建一个`__size`字节的共享内存。若已存在则应传入0缺省使用创建时大小
    - `shmflg` 由九个权限标志构成，它们的⽤法和创建⽂件时使⽤的`mode`标志是⼀样的
         - `| IPC_CREAT`：共享内存不存在，创建并返回；共享内存已存在，获取并返回
         - `| IPC_CREAT | IPC_EXCL`：共享内存不存在，创建并返回；共享内存已存在，出错返回。
- 返回值：成功返回⼀个⾮负整数，即该共享内存段的标识码，失败返回`-1`
- 标识码供上层用户使用
```c
#include <sys/types.h>
#include <sys/shm.h>
#include <sys/ipc.h>
void *shmat(int shmid, const void *shmaddr, int shmflg)
```
- 功能：将指定的共享内存段映射到当前进程的虚拟地址空间中
- 标识码供内核定位具体的共享内存段
- 参数：
   - `shmid` 共享内存标识码，即 `shmget` 的返回值
   - `shmaddr` 指定连接的具体地址。通常传入 `NULL / nullptr`，由操作系统内核自动选择一个合适的页对齐地址来映射
   - `shmflg` 读写权限标志位
      - `0` 缺省使用共享内存创建时权限
      - `SHM_RDONLY` 以只读方式挂接共享内存
- 返回值：成功返回一个指针，指向共享内存映射在当前进程中的首地址，失败返回 `(void *)-1`，并设置 `errno`
```c
#include <sys/types.h>
#include <sys/shm.h>
#include <sys/ipc.h>
int shmdt(const void *shmaddr)
```
- 功能：将共享内存段与当前进程脱离
- 参数：
   - `shmaddr` 被挂接的共享内存的首地址，即 `shmat` 成功时的返回值
- 返回值：成功返回 `0`，失败返回 `-1`，并设置 `errno`
```c
#include <sys/ipc.h>
#include <sys/shm.h>
#include <sys/ipc.h>
int shmctl(int shmid, int cmd, struct shmid_ds *buf)
```
- 功能：用于控制共享内存的状态，常用的场景是删除共享内存段
- 参数：
   - `shmid` 共享内存标识码
   - `cmd` 将要采取的具体控制动作，常用宏定义如下：
      - `IPC_RMID`：标记删除共享内存段，当所有关联该内存的进程都执行了 `shmdt` 断开连接后，内核才会真正释放物理内存
      - `IPC_STAT`：获取共享内存的状态，把 `shmid_ds` 结构中的数据设置为共享内存的当前关联值
      - `IPC_SET`：在进程有足够权限的前提下，把共享内存的当前关联值设置为 `buf `数据结构中给出的值
   - `buf` 指向一个保存着共享内存模式状态和访问权限的数据结构。如果只是为了删除，直接传入 `NULL/nullptr` 即可
```c
#pragma once
#include <iostream>
#include <sys/ipc.h>
#include <sys/shm.h>
#include <sys/types.h>
#include <string>

inline std::string g_pathname{"."};
inline int g_proj_id{0x66};
inline int g_default_size{4096};

class SharedMemory
{
public:
    SharedMemory(int size = g_default_size)
    : _size(size)
    , _key(0)
    , _shmid(-1) 
    , _start_addr(nullptr)
    , _is_creator(false)
    {}

    SharedMemory(const SharedMemory&) = delete;
    SharedMemory& operator=(const SharedMemory&) = delete;
    bool InitKey()
    {
        _key = ftok(g_pathname.c_str(), g_proj_id);
        if(_key < 0) 
        {
            perror("ftok");
            return false;
        }
        return true;
    }

    bool Create()
    {
        if (!InitKey()) return false;
        printf("形成键值: 0x%x\n", _key);
        _shmid = shmget(_key, _size, IPC_CREAT | IPC_EXCL | 0666); // 创建全新的shm
        if(_shmid < 0)
        {
            perror("shmget create");
            return false;
        }
        _is_creator = true;
        printf("形成shmid成功: %d\n", _shmid);
        return true;
    }

    bool Get()
    {
        if (!InitKey()) return false;

        // 客户端不需要 IPC_CREAT 和 IPC_EXCL，直接拿权限即可
        _shmid = shmget(_key, 0, 0666);
        if(_shmid < 0) 
        {
            perror("shmget get");
            return false;
        }
        _is_creator = false;
        printf("获取shmid成功: %d\n", _shmid);
        return true;
    }
    bool Remove()
    {
        if (_shmid < 0) return true;
        int n = shmctl(_shmid, IPC_RMID, nullptr);
        if(n < 0)
        {
            perror("shmctl");
            return false;
        }
        std::cout << "删除shm成功" << std::endl;
        return true;
    }
    bool Attach()
    {
        _start_addr = shmat(_shmid, nullptr, 0); // 这里标志位设为0则缺省使用共享内存创建时权限
        if(_start_addr == (void*)-1)
        {
            perror("shmat");
            _start_addr = nullptr;
            return false;
        }
        return true;
    }
    bool Detach()
    {
        if (_start_addr == nullptr) return true;
        int n = shmdt(_start_addr);
        if(n < 0)
        {
            perror("shmdt");
            return false;
        }
        _start_addr = nullptr;
        return true;
    }
    void* GetAddr() const 
    {
        return _start_addr;
    }
    ~SharedMemory() 
    {
        Detach();
        if (_is_creator) 
        {
            Remove();
        }
    }
private:
    int _size;
    key_t _key;
    int _shmid;
    void* _start_addr;
    bool _is_creator;
};  
```
现在我们要用上面封装的接口进行进程间通信
Server 进程 - 写数据
```cpp
#include "Shm.hpp"
#include <iostream>
#include <memory>
#include <unistd.h>
#include <cstring>

int main() // server.cc
{
    // 1. 实例化对象
    auto shm = std::make_unique<SharedMemory>();
    // 2. 创建并挂载
    shm->Create();
    shm->Attach();
    // 3. 获得挂载进虚拟地址空间的共享内存的具体起始地址
    void* addr = shm->GetAddr();
    // 4. 写入数据
    strcpy((char*)addr, "i am process A");
// while(1) {}
}
```
Client 进程 - 读数据
```cpp
#include "Shm.hpp"
#include <iostream>
#include <memory>

int main() // client.cc
{
    // 1. 实例化对象
    auto shm = std::make_unique<SharedMemory>();
    // 2. 获取并挂载
    shm->Get();
    shm->Attach();
    // 3. 获得挂载进虚拟地址空间的共享内存的具体起始地址
    void* addr = shm->GetAddr();
    // 4. 读取并打印数据
    std::cout << static_cast<char*>(addr) << std::endl;
    return 0;
}
```
### System V 消息队列
TO DO
### System V 信号量
TO DO
