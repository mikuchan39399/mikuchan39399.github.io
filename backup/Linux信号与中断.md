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

`SIGINT`信号被阻塞之后就算不捕捉它，也无法到达信号递达的状态，也就退出不了进程了

捕捉信号也可以用一组`oop`一些的接口：
```c
#include <iostream>
#include <unistd.h>
#include <signal.h>

void handler(int signo)
{
    std::cout << "获取到一个信号: " << signo << std::endl;
}

int main()
{
    struct sigaction act, old_act;
    act.sa_handler = handler;
    act.sa_flags = 0;
    sigemptyset(&(act.sa_mask));
    sigaction(SIGTERM, &act, &old_act);
    while(1)
    {
        std::cout << "我是一个进程: " << getpid() << std::endl;
        sleep(1);
    }
}
```
这个`sa_mask`成员比较有意思
这个信号集让对应置1信号被处理时自动变为阻塞状态，防止信号被递归处理

### 信号处理
**信号处理是异步的！**
也就是说，一个进程收到信号的时候，他不一定会马上去执行对应的`handler`函数
那它什么时候才去执行？
1. 此时程序处于**用户态**愉快地执行着，此时因为神秘的原因程序进入**内核态**
2. 程序现在处于**内核态**，处理完对应任务之后，顺手检查一下看看有没有信号需要处理的，如果没有捕捉信号采用自定义处理办法，忽略和默认行为操作系统知根知底，顺手执行完后返回**用户态**原先的位置继续执行下去
3. 如果用户自定义了信号处理函数，万一这个函数有问题怎么办，可不敢让它在**内核态**执行，这里什么权限都有，操作系统要回到**用户态**执行这个函数
4. 回到**用户态**，此时程序去执行那个自定义函数，在**用户态**找不到原来的指令在哪，需要返回**内核态**
5. **内核态**中有一个`sys_sigretun()`函数，拥有足够的权限让程序返回用户态后从主控制流程中上次被中断的地方继续往下执行

<img width="649" height="221" alt="Image" src="https://github.com/user-attachments/assets/9dfbd060-3bb8-4210-beb9-516a41369ae5" />

这就是一个无限形的流程，节点在**内核态**
## 中断---操作系统到底是怎么运行的
在 CPU 日常运行各种程序的时候，鼠标点了一下，键盘按了一下，网卡收到了数据，等等不受预期的行为，都会打断 CPU 的动作，迫使 CPU 放下手中的事情**进入内核态**去执行其他任务，这种行为被称作中断，也是操作系统本身能够运作的重要基础
### 硬件中断/外中断
在键盘上按个快捷键是怎么让操作系统给进程发信号的？
这就是一种硬件中断！
我们电脑上的各种外设设备，都连接着一个名叫**中断控制器**的硬件结构，用来管理杂乱的外设请求。操作系统中存在着一个名叫**中断向量表**`IDT`的指针数组，在操作系统开机时就存在于内存中
一次完整的硬件中断的流程是这样的：
1. 外设就绪：键盘被按下，鼠标被点击
2. 发起中断：外设向中断控制器发起中断，中断控制器管理好外设的中断请求，决定优先级
3. 通知 CPU：CPU 上的 `INTR`引脚接收中断控制器发来的高电平信号
4. CPU 回应：在 CPU 回应中断控制器时，中断控制器把这个中断对应的中断向量号（一个 8 位的整数编号）通过数据总线扔给 CPU
5. CPU 保护现场：保存中断前的上下文，以便中断结束后继续执行任务
6. CPU 陷入内核态，根据中断号，去中断向量表中查表，执行对应的中断处理程序`ISR`，以回应不同的中断需求
7. CPU 执行完毕，恢复现场中断结束，继续执行之前的工作
 
#### 时钟中断
之前提到过，进程检查信号需要 CPU 先陷入内核态，那不能没有外设就绪就不检查信号了吧？
操作系统来调度进程，可是谁来调度操作系统自己？
CPU 中内置了一块称作**时钟源**的硬件，它可以周期性地以固定时间间隔不断通过中断向量表或可编程间隔定时器，向 CPU 发起中断，这被称作**时钟中断**
> [!Note]
> CPU 主频 和 时钟中断频率并不等价
> CPU 主频 = 外频（时钟源频率 / BCLK） × 倍频（Multiplier）
> 时钟源又叫晶振，只要给它通电，它就会产生极其稳定、规律的电脉冲信号，通常不高，比如 100MHz
> 时钟源的作用就是给整个主板上的各个组件提供一个统一的“基准时间尺度”，确保大家在一个节拍下协同工作，有点像乐队中的鼓手
> CPU 内部集成了一个名叫 锁相环（PLL）的电路，他能放大n倍时钟源频率，这之后才是 CPU 的主频
> 时钟中断并不是 CPU 自己凭空产生的。主板上还有一个专门的可编程硬件定时器（如 PIT 或 APIC Timer），它同样接收晶振脉冲信号。
> 定时器会根据设定好的计数值不断倒数，每倒数到 0，就会向 CPU 发送一次硬件中断信号（IRQ 0）。
> 操作系统在启动时，会配置这个定时器的倒数周期，从而决定了时钟中断频率（例如 Linux 内核中的 HZ 宏，通常配置为 250Hz 或 1000Hz）。

> [!important] 
> CPU 主频决定了指令执行有多快，而时钟中断频率由操作系统规定

有了周期性的时钟中断，CPU 就能每隔一段时间自动陷入内核态，完成基本的信号检测
而时钟中断对应的中断向量，也就是对应时钟中断的中断处理程序，就是完成操作系统基本生命活动的代码集合
> [!important]
**操作系统的本质：一个由中断驱动的“死循环”**
操作系统在开机初始化完毕后，其实就进入了一个静静等待事件发生的无限循环。它能够持续掌控全局，不被任何跑飞的用户程序独占 CPU，最核心的底气就来源于周期性的时钟中断。
硬件定时器就像维持系统生命特征的“心跳”，每隔一段固定时间就强制触发一次中断。这迫使 CPU 暂停当前正在执行的用户程序，自动陷入内核态。
随后，CPU 去中断向量表中寻找时钟中断对应的`ISR`并执行。这段内核代码承载了操作系统最基础的生命活动：它不仅会顺手检查并处理当前进程的未决信号，还会**更新系统时间、扣减当前进程的时间片，并在时间片耗尽时触发进程调度。没有时钟中断，整个操作系统的运转就会陷入停滞**。
```c
void main(void) /* 这⾥确实是void，并没错。 */
{ /* 在startup 程序(head.s)中就是这样假设的。 */
...
/*
* 注意!! 对于任何其它的任务，'pause()'将意味着我们必须等待收到⼀个信号才会返
* 回就绪运⾏态，但任务0（task0）是唯⼀的意外情况（参⻅'schedule()'），因为任
* 务0 在任何空闲时间⾥都会被激活（当没有其它任务在运⾏时），
* 因此对于任务0'pause()'仅意味着我们返回来查看是否有其它任务可以运⾏，如果没
* 有的话我们就回到这⾥，⼀直循环执⾏'pause()'。
*/
    for (;;) 
    pause();
} // end main
```
### 内中断
内中断，又叫做异常，相比于异步的外中断，是同步触发的，分为这三类：
- `Trap` 陷阱/软中断
- `Fault` 故障
- `Abort` 终止
#### Trap
在代码中主动执行的，能让 CPU 陷入内核的指令，被称作 `Trap`
一般有 `syscall`  `int 0x80`  `int 3` 这三种
在32位机器上，一般使用 `int 0x80`，意思就是，去执行中断向量表中第 `128`号 中断向量对应的代码
在64位机器上，一般使用 `syscall`，它不去查中断向量表，而是直接去**模型特定寄存器**`MSR`中读出代码地址，更快

**揭秘系统调用的幕后**
我们平时用的**系统调用**，实际上并不是真正的系统调用接口，而是C语言封装好的系统调用库
内核中存在一个**系统调用表**，也就是函数指针数组，它们指向真正的系统调用
在我们调用系统库函数时：
1. C库会帮我们找到系统调用名对应的系统调用号，也就是那个指针数组下标
2. 把这个系统调用号写入**通用寄存器**中，`mov eax, n`
3. 然后调用 `syscall` 或 `int 0x80` 陷入内核态
4. 对应的中断执行代码则会拿到寄存器中的系统调用号，去查系统调用表并执行
> 所以说系统调用的实现借助了 `Trap` 软中断