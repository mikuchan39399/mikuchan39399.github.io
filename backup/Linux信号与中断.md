## Ctrl + C 干了什么
平时我们运行一些死循环程序时，可以在命令行终端上用`Ctrl + C` 快捷键结束进程。
实际上，这个快捷键让键盘外设控制 CPU 让操作系统往目标进程发送`SIGINT`**信号**以杀死进程，这里有一个 API 能让我们验证一下...
```c
#include <signal.h>
typedef void (*sighandler_t)(int);
sighandler_t signal(int signum, sighandler_t handler);
```
- 作用：修改目标信号的处理回调函数
- 参数
   - `singnum` 信号编号
   - `handler` 信号处理函数
- 返回值 修改之前的旧信号处理函数的函数指针
```cpp
#include <iostream>
#include <csignal>
void handler(int signo)
{
    std::cout << "捕捉到" << signo << "号信号\n"; 
}
int main()
{
    signal(SIGINT, handler);
    while(1);    
    return 0;
}
```
我用这个接口捕捉了`SIGINT`信号，让我们看看现在用`Ctrl + C`还能不能杀掉这个进程

<img width="1092" height="260" alt="Image" src="https://github.com/user-attachments/assets/817ccb36-d25c-4e67-95f5-763f32b2682a" />

当使用`Ctrl + C`杀不掉某个进程的时候，一种可能是程序捕捉了`SIGINT`，还有可能是这个进程不是**前台进程**，而前台进程只有一个，键盘只能给前台进程发信号，⼀个命令后⾯加个&可以放到后台运⾏,这样 Shell 不必等待进程结束就可以接受新的命令启动新的进程。

## 信号机制
可以用`kill -l` 命令查看所有信号

<img width="936" height="401" alt="Image" src="https://github.com/user-attachments/assets/3479ebe3-d43f-4f5b-bef9-6e311180c212" />

> 为什么没有32, 33号信号？貌似是因为某些历史遗留原因

每个信号名都有一个宏，表示信号名对应的信号编号
编号34以上是实时信号，咱们在这儿不讨论实时信号，可以用`man 7 signal`查询各个信号的产生条件和默认处理动作
信号的默认处理动作有三种：
- 忽略 `SIG_IGN`
- 默认 `SIG_DFL`
- 捕捉，提供相应自定义信号处理函数
> 其实SIG_DFL和SIG_IGN就是把0,1强转为函数指针类型
### 信号产生
信号产生有以下几种方式
1. 外设，例如刚才键盘上的 `Ctrl + C`
2.  命令行工具：`kill -信号名 pid` 可以往指定进程发送对应信号
3.  程序内函数： 
```c
#include <sys/types.h>
#include <signal.h>
int kill(pid_t pid, int sig); // 往指定进程发送信号

#include <signal.h>
int raise(int sig); // 往当前进程发送信号

#include <stdlib.h>
void abort(void); // 让当前进程接受信号而 <异常> 终止
// 就像exit函数⼀样,abort函数总是会成功的,所以没有返回值。
```
4. 软件产生信号
```c
#include <unistd.h>
unsigned int alarm(unsigned int seconds);
```
- 作用：设定一个闹钟，让内核在 seconds 秒后给当前进程发送 SIGALRM 
- 返回值：
   - 此闹钟之前没有运行时闹钟，返回 0
   - 否则返回运行时闹钟剩余触发秒数并重置闹钟状态为当前调用
5. 硬件异常产生信号
硬件因为某些原因产生异常后，被硬件以某种方式检测到并通知操作系统内核，让内核向当前进程发送适当信号，例如：
- 除以0 - `CPU` 上的 `ALU` 运算单元产生异常，操作系统发送`SIGFPE`信号
- 野指针/空指针/非法内存地址 - `MMU`内存管理单元产生异常，操作系统发送`SIGSEGV`信号

### 信号传递
实际执行信号处理动作被称为信号递达`Delivery`
信号从产生到递达之间的状态，被称为信号未决`Pending`
信号可以被进程选择性阻塞
被阻塞的信号会一直处于未决状态，直到进程解除对此信号的阻塞，才能信号递达
阻塞不同于忽略，忽略本身就是一种信号递达
每个进程当中都有两个位图和一个函数指针数组，分别为：
- `block` 位图
- `pending` 位图
- `handler`  `sighandler_t`类型函数指针数组
这些位图的数据类型是`sigset_t`，被称作信号集，`block`信号集也被称为信号屏蔽字
信号编号就是这两个信号集和函数指针数组的下标，`SIG_IGN`和`SIG_DFL`要被强转成`sighandler_t`类型放进这个数组中
这两个信号集用于标识进程中每个信号的状态
有一些可以操纵信号集的函数：
```c
#include <signal.h>
int sigemptyset(sigset_t *set); //清空信号集为全0
int sigfillset(sigset_t *set);  // 填满信号集为全1
int sigaddset(sigset_t *set, int signo); //将信号集的目标信号编号标志位置为1
int sigdelset(sigset_t *set, int signo); // 将信号集的目标信号编号标志位置为0
int sigismember(const sigset_t *set, int signo); // 判断目标信号标志位在信号集是否为1
int sigpending(sigset_t *set); // 读取当前未决信号集，通过set输出型参数传出
```
比较重要的一个：
```c
#include <signal.h>
int sigprocmask(int how, const sigset_t *set, sigset_t *oset);
```
作用：读取或更改进程的信号屏蔽字/阻塞信号集
参数：
- set 作用于进程的信号屏蔽字
- oset 之前的信号屏蔽字
- how
   - SIG_BLOCK 按位或
   - SIGUNBLOCK 按位与 (~set)，解除阻塞的信号
   - SIG_SETMASK 覆盖
 
以下代码演示了阻塞的信号只能待在未决信号集里吃灰
```c
#include <iostream>
#include <csignal>
#include <unistd.h>
#include <functional>
#include <sys/wait.h>
#include <vector>
#include <sys/types.h>

void PrintPending(sigset_t& pending)
{
    std::cout << "\n[ sigpending list: ";
    for(int i = 31; i; i--)
    {
        if(sigismember(&pending, i))
        {
            std::cout << "1";
        }
        else
        {
            std::cout << "0";
        }
    }
    std::cout << " ]" << "\r";
}

int main()
{
    sigset_t block, old_block;
    sigemptyset(&block); sigemptyset(&old_block);
    sigaddset(&block, SIGINT);
    sigprocmask(SIG_SETMASK, &block, &old_block);
    while(true)
    {
        sigset_t pending;
        sigemptyset(&pending);
        sigpending(&pending);
        PrintPending(pending);
        sleep(1);
    }
    return 0;
}
```

<img width="1174" height="357" alt="Image" src="https://github.com/user-attachments/assets/1f23710e-b26a-48b1-8885-8f588e0d973a" />

可
### 信号处理
## 中断
### 硬件中断
### 软中断
### 异常