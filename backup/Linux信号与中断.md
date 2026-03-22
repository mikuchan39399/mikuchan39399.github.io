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

## 信号机制
### 信号产生
### 信号传递
### 信号处理
### 信号相关 API
## 中断
### 硬件中断
### 软中断
### 异常