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

当使用`Ctrl + C`杀不掉某个进程的时候，一种可能是程序捕捉了`SIGINT`，还有可能是这个进程不是**前台进程**，而前台进程只有一个

## 信号机制
### 信号产生
### 信号传递
### 信号处理
### 信号相关 API
## 中断
### 硬件中断
### 软中断
### 异常