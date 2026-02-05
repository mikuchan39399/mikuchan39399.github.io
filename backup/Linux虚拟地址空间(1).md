## 回顾 C 语言程序地址空间
> 从低地址开始，到高地址……
> 首先，我们来看三个最基本的静态区域！
### 1. 正文代码区 (.text / .rodata)
该区域对应于编译后的机器指令
- 它存放着：
   - 机器代码：也就是 CPU 执行的二进制指令，高级语言中所写的 if-else / while / for 与函数调用等逻辑，编译后都在这里
   - 只读常量：存放 字符串字面量 以及 const 修饰的 全局变量
- 特点：
   - 只读：防止程序运行修改指令
   - 共享：一个程序创建子进程的话，父子进程之间共享这段代码段，节省内存
   - 位置：位于程序地址空间最底层
```cpp
// --- (.text / .rodata) ---
const int g_const = 100;
char *str = "Hello"; //实际上，字符串字面量"Hello"存在于.rodata中，指针变量 str 作为普通变量存在于其他区域中
char arr[] = "Hello"; // "Hello" 也是字面量，但把 arr[ ] 看作变量，会在变量对应的内存区内开辟一块数组空间拷贝进去
// 此时修改 arr[0] 是合法的，修改 str[0] 会 段错误。
```
> 在具体的操作系统中，.text 通常专门存放机器代码，.rodata 专门存放只读常量

### 2. 初始化数据区 (.data)
这个区域存放的是程序启动时就已经确定数值的变量
- 它存放着：
   - 显式初始化为非零值的 全局变量
   - 显式初始化为非零值的 static 修饰的静态变量
- 特点：
   - 读写：程序运行过程中允许对此区域进行读取修改
   - 占用文件空间：这些变量初始值存在于可执行文件 (exe / elf)，程序运行时将它们从磁盘拷贝至内存

```cpp
// --- (.data) ---
int g_var = 10;
static int s_var = 20;
```
 ### 3. 未初始化数据区 (.bss)
这个区域的名字来源于汇编语言时代的 "Block Started by Symbol"
- 它存放着：
   - 显式初始化为零值 或 没有初始化的 全局变量
   - 显式初始化为零值 或 没有初始化的 static 修饰的 静态变量
- 特点：
   - 自动清零： 操作系统在加载程序时，会将这块内存全部置为 0。这也是为什么全局变量不赋值默认是 0。
   - 不占文件空间： 这是它存在的最大意义。比如我定义了一个 int arr[10000]; 的全局数组且没赋值，可执行文件不会因此变大 40KB。可执行文件只需要记录“这里需要预留 40KB 的空间”这个信息即可。

```cpp
// --- (.bss) ---
int g_bss;
int g_zero = 0; 
static int s_bss;
```
> 接下来是与用户关联较大的...
### 堆栈
堆区与栈区相对而生，口头上提到堆栈区时没有特别说明的话默认指栈区
严谨讨论时最好把他们两个区分开
堆区从堆栈区低位开始延展至高位，栈区从堆栈区高位开始延展至低位

#### 栈区与函数栈帧
当一个函数被调用时，编译器生成的指令让 CPU 移动栈指针，从而在栈上划出一段空间，形成函数栈帧（OS 提供栈映射与保护）
事实上，栈区就是由一个个函数栈帧堆叠而成的
它是一个复杂的结构体，
- 包含：
   - 函数中定义的局部变量 (不包括静态变量)
   - 调用函数时传入的参数 (这个也有可能在寄存器之间传递不放进栈帧)
   - 返回地址：函数调用完毕之后程序应当跳转到哪里继续执行
   - 旧的栈底指针 EBP / RBP：用于找到**在栈区弹出该栈帧时应该回到的上一个栈帧**的底部位置
   - 保存的寄存器：函数执行过程中需要用到寄存器，为了不覆盖调用者的数据，需要先把寄存器里的(属于调用者)的旧值存在栈帧里，用完了再恢复 
   > 一般操作系统会给栈区规定好一个较小的空间大小，例如8MB，栈区超过这个范围则会栈溢出
#### 堆区与用户内存管理
堆区，
- 它存放：
   - 程序员使用高级语言的各种接口向操作系统申请的内存，例如 C 的malloc和 C++ 的 new
- 特点：
   - 由程序员显式申请/释放，不使用 free 等接口释放内存的话会导致内存泄露
   - 空间相较与栈区会大很多
 > 堆区和栈区两面包夹芝士中间还有一块空白区，被称作共享区 (mmap 映射区)，用于加载动态库(.so / .dll) 或进行文件映射
 > 栈区的更高位还有一块用户空间专门用于存放启动程序时传入的命令行参数与环境变量，再往上就是留给内核的部分了
```acsii
高地址
┌─────────────────┐
│  内核空间        │
├─────────────────┤
│  环境变量/参数   │
├─────────────────┤
│  栈区 (↓)       │
│                 │
│  (空白区/mmap)  │
│                 │
│  堆区 (↑)       │
├─────────────────┤
│  .bss           │
├─────────────────┤
│  .data          │
├─────────────────┤
│  .rodata        │
├─────────────────┤
│  .text          │
└─────────────────┘
低地址

```
### 引入虚拟地址空间
> 有句话告诉我们，操作系统是对于计算机硬件的最底层抽象，学习 C语言时接触的这些地址与指针，真的是计算机物理上确确实实的地址吗......当然不是~
```cpp
#include <iostream>
#include <sys/types.h>
#include <unistd.h>

int gval = 0;

int main()
{
    pid_t id = fork();
    if(id < 0)
    {
        perror("fork");
        return 1;
    }
    else if(id == 0) //child
    {
        pid_t pid = getpid();
        pid_t ppid = getppid();
        while(1)
        {
            printf("子进程, pid: %d, ppid: %d, gval: %d, &gval: %p\n", pid, ppid, gval, &gval);
            sleep(1);
            gval++;
        }
    }
    else //fa
    {
        pid_t pid = getpid();
        pid_t ppid = getppid();
        while(1)
        {
            printf("父进程, pid: %d, ppid: %d, gval: %d, &gval: %p\n", pid, ppid, gval, &gval);
            sleep(1);
        }
    }
    return 0;
}
```

<img width="748" height="585" alt="Image" src="https://github.com/user-attachments/assets/5f8c726b-73be-4b56-9592-7a3ae6855f90" />

如果C语言中的地址代表物理地址的话，物理地址相同的变量就不可能拥有不一样的两个值对吧
证明操作系统绝对是对硬件上的内存做了抽象的，这层抽象在 OS 学科里就被叫做虚拟地址空间
在Linux中被称作进程虚拟地址空间
### mm_struct
```cpp
1    struct mm_struct
2    {
3        /*...*/
4        struct vm_area_struct *mmap;        /* 指向虚拟区间(VMA)链表 */
5        struct rb_root mm_rb;               /* red_black树 */
6        unsigned long task_size;            /*具有该结构体的进程的虚拟地址空间的大小*/
7        /*...*/
8        // 代码段、数据段、堆栈段、参数段及环境段的起始和结束地址。
9        unsigned long start_code, end_code, start_data, end_data;
10       unsigned long start_brk, brk, start_stack;
11       unsigned long arg_start, arg_end, env_start, env_end;
12       /*...*/
13   }
```
操作系统会给每个进程画饼，意思是每个进程都能看到全局的虚拟地址空间，但是进程间怎么配合使用肯定由操作系统来宏观调控
在 task_struct 中存在一个结构体指针 mm_struct*，指向对应进程唯一的内存描述符
```cpp
struct task_struct
{
/*...*/
struct mm_struct       *mm; //对于普通的用户进程来说该字段指向他的虚拟地址空间的用户空间部分，对于内核线程来说这部分为NULL。
struct mm_struct       *active_mm; // 该字段是内核线程使⽤的。当该进程是内核线程时，它的mm字段为NULL，表⽰没有内存地址空间，
可也并不是真正的没有，这是因为所有进程关于内核的映射都是一样的，内核线程可以使用任意进程的地址空间。
/*...*/
}
```
 和进程控制块一样，成套的内存描述符也需要被管理
 当虚拟区较少时采取单链表，由mmap指针指向这个链表
 当虚拟区间多时采取红⿊树进⾏管理，由mm_rb指向这棵树
#### 虚拟内存区域 VMA
Linux 使用 vm_area_struct 结构体来表示一个独立的虚拟内存区域，它们就存在于 mm_struct 中
一个 mm_struct 中拥有多个 vm_area_struct ，用来表示内部机制与功能都不尽相同的各个虚拟内存区
```cpp
1    struct vm_area_struct {
2        unsigned long vm_start; //虚存区起始
3        unsigned long vm_end;   //虚存区结束
4        struct vm_area_struct *vm_next, *vm_prev;   //前后指针
5        struct rb_node vm_rb;   //红黑树中的位置
6        unsigned long rb_subtree_gap;
7        struct mm_struct *vm_mm;    //所属的 mm_struct
8        pgprot_t vm_page_prot;
9        unsigned long vm_flags;     //标志位
10       struct {
11           struct rb_node rb;
12           unsigned long rb_subtree_last;
13       } shared;
14       struct list_head anon_vma_chain;
15       struct anon_vma *anon_vma;
16       const struct vm_operations_struct *vm_ops;  //vma对应的实际操作
17       unsigned long vm_pgoff;     //文件映射偏移量
18       struct file * vm_file;      //映射的文件
19       void * vm_private_data;     //私有数据
20       atomic_long_t swap_readahead_info;
21   #ifndef CONFIG_MMU
22       struct vm_region *vm_region;        /* NOMMU mapping region */
23   #endif
24   #ifdef CONFIG_NUMA
25       struct mempolicy *vm_policy;        /* NUMA policy
```
这里讲几个比较核心的~
- vm_next / vm_prev / vm_rb : 前面刚说的对于 mm_struct 进行管理的方式就是通过 vm_area_struct 中的前后指针与 vm_rb 进行细颗粒度管理的
- vm_start / vm_end 这里划定了虚拟内存区在虚拟地址下的区域范围
- vm_page_prot 定义了虚拟内存区的权限，这就能解释为什么上文中的虚拟内存区存在对应的权限了，这是内核级别的权限而非语法应用层面

### 虚拟地址空间的意义
- 保护物理内存，防止程序弄坏他不应该触及到其他各种数据
- 对进程管理与内存管理之间的解耦合，虚拟内存的加载不依赖物理内存
- 让进程视角实现了内存的有序


> 为什么同一个虚拟地址，在不同进程里能指向不同的物理内存？
> 答案是：每个进程都有自己的页表。
> 页表就像一本"地址翻译字典"，CPU 通过它把虚拟地址翻译成物理地址。不同进程的字典不一样，所以同一个虚拟地址0x601040 在进程A里翻译成物理地址 0x12345000，在进程B里翻译成 0x67890000。

具体怎么翻译的？我们下一篇见。


