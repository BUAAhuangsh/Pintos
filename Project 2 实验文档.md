# Project 2 实验文档

[TOC]



## 团队 那遺矢dé圊舂|ゞ

### 基本信息

| 姓名   | 学号     | Git 用户名   |
| ------ | -------- | ------------ |
| 黄森辉 |          |              |
| 陈逸然 | 19231005 | aurora-cccyr |
| 吴浩华 | 19231023 |              |
| 赵晰莹 | 19231235 | wellmslouis  |

### 每位组员的主要工作内容

### Git 相关

[Pintos 的 Git 项目地址](https://github.com/BUAAhuangsh/Pintos.git)

### 参考资料

1. []()
2. []()

## 实验要求

本次实验要求我们完成在运行用户程序时，操作系统对内存、调度等的管理。

大致要求可分为以下方面:

1.参数传递和打印进程终结信息

每当一个用户进程因为该进程调用`exit`或其它原因而结束时，需要打印该进程的进程名和退出码（`exit code`）；拓展process_execute函数功能，使其能够给新进程传递参数（将文件名和参数用空格分离）

2.系统调用

实现系统调用，需要根据系统提供的一组完成底层操作的函数集合，由用户程序通过中断调用，系统根据中断向量表和中断服务号确定函数调用，调用相应的函数完成相应的服务。

## 需求分析

## 设计思路

![image-20211207105612405](../AppData/Roaming/Typora/typora-user-images/image-20211207105612405.png)

### 系统调用

系统调用是由系统提供的一组完成底层操作的函数集合，由用户程序通过中断调用，系统根据中断向 量表和中断服务号确定函数调用，调用相应的函数完成相应的服务。其主要目的，是在中断的时候根据系统调用号执行函数。大致流程为根据中断帧的指针，通过 esp 找到用户程序调用号和参数，再根据系统调用号，转向系统调用功能函数，并将参数传入，之后执行系统调用具体功能（如创建文件），最后执行完，将返回值写入 eax中。

在syscall.c中：

```c
void
syscall_init (void) 
{
  intr_register_int (0x30, 3, INTR_ON, syscall_handler, "syscall");
}
```

当用户程序调用系统调用以后，会激发30号中断，通过中断向量指向并运行syscall_handler（）函数。
在syscall_handler中，应该需要根据系统调用号来调用不同的系统调用。

```c
static void
syscall_handler (struct intr_frame *f UNUSED) 
{
  printf ("system call!\n");
  thread_exit ();
}
```

由参数传递部分可以知道，intr_frame 的 esp 栈顶里保存着中断服务号 NUMBER，并且在栈顶的后面若干个字节保存了调用的其他参数。我们需要把syscall 设计为一个调用其他函数的流程控制器，保持这个函数的过程清晰。

在lib/user/syscall.h下可以看到本次系统调用的用户需求和传递参数：

```c
void halt (void) NO_RETURN;
void exit (int status) NO_RETURN;
pid_t exec (const char *file);
int wait (pid_t);
bool create (const char *file, unsigned initial_size);
bool remove (const char *file);
int open (const char *file);
int filesize (int fd);
int read (int fd, void *buffer, unsigned length);
int write (int fd, const void *buffer, unsigned length);
void seek (int fd, unsigned position);
unsigned tell (int fd);
void close (int fd);
```

在lib/syscall­nr.h中则提供了对应的服务

```c
 /* Projects 2 and later. */
 SYS_HALT,                   /* Halt the operating system. */
 SYS_EXIT,                   /* Terminate this process. */
 SYS_EXEC,                   /* Start another process. */
 SYS_WAIT,                   /* Wait for a child process to die. */
 SYS_CREATE,                 /* Create a file. */
 SYS_REMOVE,                 /* Delete a file. */
 SYS_OPEN,                   /* Open a file. */
 SYS_FILESIZE,               /* Obtain a file's size. */
 SYS_READ,                   /* Read from a file. */
 SYS_WRITE,                  /* Write to a file. */
 SYS_SEEK,                   /* Change position in a file. */
 SYS_TELL,                   /* Report current position in a file. */
 SYS_CLOSE,                  /* Close a file. */
```

使用syscall_handler完成函数分配时有两种思路，一种用switch-case寻找对应的函数，另一种事先在syscall_init中声明函数指针数组，之后根据系统调用号选择函数。为了整体的简洁明了，这里我们选择第二种方法。

```c
# define max_syscall 20
// 系统调用数组的实现
static void (*syscalls[max_syscall])(struct intr_frame *);

void
syscall_init (void)
{
  //初始化30号中断，使其指向syscalls_handler
  intr_register_int (0x30, 3, INTR_ON, syscall_handler, "syscall");
  syscalls[SYS_HALT] = &sys_halt;
  syscalls[SYS_EXIT] = &sys_exit;
  syscalls[SYS_EXEC] = &sys_exec;
  syscalls[SYS_WAIT] = &sys_wait;
  syscalls[SYS_CREATE] = &sys_create;
  syscalls[SYS_REMOVE] = &sys_remove;
  syscalls[SYS_OPEN] = &sys_open;
  syscalls[SYS_WRITE] = &sys_write;
  syscalls[SYS_SEEK] = &sys_seek;
  syscalls[SYS_TELL] = &sys_tell;
  syscalls[SYS_CLOSE] =&sys_close;
  syscalls[SYS_READ] = &sys_read;
  syscalls[SYS_FILESIZE] = &sys_filesize;
}
```

我们知道系统调用号隐藏在f->esp中，在syscall_handler中获得系统调用号后检查其是否合法，如果合法，就执行对应函数。当pintos进行系统调用的时候，它先会调用静态函数syscall_handler (struct  intr_frame *f )，Intr_frame是指向用户程序寄存器。这里如果不合法，先将进程的结束状态改为-1，之后调用 thread_exit()终止进程，由 thread_exit()调用 process_exit()，释放进程所占用的资源。

```c
static void
syscall_handler (struct intr_frame *f)
{
  //得到系统调用的类型（系统调用号）（int）
  int type = * (int *)f->esp;
  if(type <= 0 || type >= max_syscall){
    thread_current()->st_exit = -1;//将进程的结束状态改为-1
    thread_exit ();//退出进程
  }
  syscalls[type](f);
}
```

此外，在基础的系统调用之上，我们有必要采取一系列措施，在用户程序产生不当操作时，终止用户进程。那么，对于用户提供的指针，在解引用前，必须先检查指针不为空、指针指向的是自己的用户虚存地址空间。

> 在访问错误中，我们大致可以将错误分为以下几种：
>
> 1.用户访问系统内存，对该区域的写操作有可能导致操作系统异常。
>
> 2.用户访问尚未被映射或不属于自己的用户虚存地址空间
>
> 3.参数无效导致文件加载失败
>
> 4.向可执行文件写入

其中1,2,3的情况，我们可以用一个统一的check_ptr函数进行安全检查。

> 目前的 pintos 有一个简单的虚存映射机制，大致的流程如下：
>
> 1)规定将实存一分为二，低空间分配给内核，高空间分配给用户；
>
> 2)系统初始化时，将指定 PHYS_BASE，高于 PHYS_BASE 的虚存地址空间分配给内核；
>
> 3)在初始化用户进程时，指定一块虚存地址空间给该进程，具体的页表信息保存在该进程对应线
>
> 程的结构体 struct thread 中，其成员变量名为 pagedir; 

如果访问尚未被映射的地址空间，会造成 page_fault，引发系统崩溃，因此，我们首先检查用户指针是否指向PHYS_BASE。

```c
/* 在用户虚拟地址 UADDR 读取一个字节。
   UADDR 必须低于 PHYS_BASE。
   如果成功则返回字节值，如果
   发生段错误则返回 -1 。*/ 
static int 
get_user (const uint8_t *uaddr)
{
  int result;
  asm ("movl $1f, %0; movzbl %1, %0; 1:" : "=&a" (result) : "m" (*uaddr));
  return result;
}

void * 
check_ptr2(const void *vaddr)
{ 
  //确保指针指向用户地址
  if (!is_user_vaddr(vaddr))
  {
    thread_current()->st_exit = -1;
    thread_exit ();
  }
  //确保页表不为空
  void *ptr = pagedir_get_page (thread_current()->pagedir, vaddr);
  if (!ptr)
  {
    thread_current()->st_exit = -1;
    thread_exit ();
  }
  //检查页表中的每一项
  uint8_t *check_byteptr = (uint8_t *) vaddr;
  for (uint8_t i = 0; i < 4; i++) 
  {
    if (get_user(check_byteptr + i) == -1)
    {
      thread_current()->st_exit = -1;
      thread_exit ();
    }
  }

  return ptr;
}
```

为了保证文件访问参数的有效性，我们在syscall_handler中检测第一个参数的合法性

```c
static void
syscall_handler (struct intr_frame *f)
{
  int * p = f->esp;
  //检查第一个参数
  check_ptr2 (p + 1);
  //得到系统调用的类型（系统调用号）（int）
  int type = * (int *)f->esp;
  if(type <= 0 || type >= max_syscall){
    thread_current()->st_exit = -1;//将进程的结束状态改为-1
    thread_exit ();//退出进程
  }
  syscalls[type](f);
}
```

进程方面的系统调用主要涉及`lib/syscall­nr.h`中前四个服务，对应的我们需要写四个函数：

```c
void sys_halt(struct intr_frame* index);
void sys_exit(struct intr_frame* index);
void sys_exec(struct intr_frame* index); 
void sys_wait(struct intr_frame* index);
```

`sys_halt`是关机函数；

```c
shutdown_power_off();
```

`sys_exit`是退出进程函数，在`thread.c`中新增退出码`st_exit`以供后续使用；

```c
thread_current()->st_exit=*(((int*)index->esp)+1);//为当前线程保存退出码
thread_exit();
```

`sys_exec`是开始另一程序的函数，调用了参数传递拟写的`process_execute`函数；

```c
index->eax = process_execute((char*)(index->esp));
```

`sys_wait`是等待子进程函数，根据需求，我们还需要在`process.c`文件中补写`process_wait`函数；

```c
index->eax = process_wait(((int*)index->esp)+1);
```

## 重难点讲解

## 用户手册

## 测试报告

**用户访问系统内存测试点如下：**

tests/userprog/read­bad­ptr 

tests/userprog/bad­write2 

tests/userprog/bad­jump2 

**用户访问尚未被映射或不属于自己的用户虚存地址空间测试点如下：**

tests/userprog/sc­bad­arg 

tests/userprog/sc­bad­sp 

tests/userprog/sc­bad­arg 

tests/userprog/open­bad­ptr 

tests/userprog/read­bad­ptr 

tests/userprog/write­bad­ptr 

tests/userprog/exec­bad­ptr 

tests/userprog/bad­read 

tests/userprog/bad­write 

tests/userprog/bad­read2 

tests/userprog/bad­write2 

tests/userprog/bad­jump 

tests/userprog/bad­jump2 

**文件参数合法性这部分对应的测试点如下：**

tests/userprog/create­empty 

tests/userprog/open­missing 

tests/userprog/open­empty 

tests/userprog/exec­missing 

tests/userprog/wait­bad­pid 

**拒绝写入可执行文件部分对应的测试点如下：**

tests/userprog/rox­simple 

tests/userprog/rox­child 

tests/userprog/rox­multichild

## 各成员的心得体会

### Student1

### Student2

### Student3

### Student4

## 其他你认为有必要的内容 (Optional)

# Project 2 Design Document

## QUESTION 1: ARGUMENT PASSING

### DATA STRUCTURES

> A1: Copy here the declaration of each new or changed `struct` or `struct` member, global or static variable, `typedef`, or enumeration. Identify the purpose of each in 25 words or less.

### ALGORITHMS

> A2: Briefly describe how you implemented argument parsing. How do you arrange for the elements of argv[] to be in the right order? How do you avoid overflowing the stack page?
>
> A2: 简要描述你是怎么实现 Argument parsing 的。你是如何安排 argv[]中的 elements，使其在正确的顺序的？你是如何避免 stack page 的溢出的？

### RATIONALE

> A3: Why does Pintos implement strtok_r() but not strtok()?
>
> A3: 为什么 Pintos 中实现 strtok_r()而不是 strtok()？

> A4: In Pintos, the kernel separates commands into a executable name and arguments. In Unix-like systems, the shell does this separation. Identify at least two advantages of the Unix approach.
>
> A4: 在 Pintos 中，kernel 将命令分成了可执行文件的 name 以及参数。在 Unix-like 的系统中，shell 完成这部分的分隔。列举至少 2 种 Unix 这样做的好处。

## QUESTION 2: SYSTEM CALLS

### DATA STRUCTURES

> B1: Copy here the declaration of each new or changed `struct` or `struct` member, global or static variable, `typedef`, or enumeration. Identify the purpose of each in 25 words or less.

> B2: Describe how file descriptors are associated with open files. Are file descriptors unique within the entire OS or just within a single process?
>
> B2: 描述文件描述符是如何与打开文件相联系的。文件描述符是在整个中唯一还是仅在单个进程中唯一？

### ALGORITHMS

> B3: Describe your code for reading and writing user data from the kernel.
>
> B3: 描述你用来从 kernel 中读写文件的代码。

> B4: Suppose a system call causes a full page (4,096 bytes) of data to be copied from user space into the kernel. What is the least and the greatest possible number of inspections of the page table (e.g. calls to pagedir_get_page()) that might result? What about for a system call that only copies 2 bytes of data? Is there room for improvement in these numbers, and how much?
>
> B4: 假设一个系统调用造成一整页的数据(4096 bytes)从用户空间复制到 kernel。
>
> 求可能造成的最小和最大的页表的检查次数。(e.g. 对 pagedir_get_page()的调用)。如果系统调用只 copy 了 2 bytes 的数据呢？还有没有空间优化？可以优化多少？

> B5: Briefly describe your implementation of the "wait" system call and how it interacts with process termination.
>
> B5: 简要描述你"wait"系统调用的实现以及它是如何与进程停止交互的。

> B6: Any access to user program memory at a user-specified address can fail due to a bad pointer value. Such accesses must cause the process to be terminated. System calls are fraught with such accesses, e.g. a "write" system call requires reading the system call number from the user stack, then each of the call's three arguments, then an arbitrary amount of user memory, and any of these can fail at any point. This poses a design and error-handling problem: how do you best avoid obscuring the primary function of code in a morass of error-handling? Furthermore, when an error is detected, how do you ensure that all temporarily allocated resources (locks, buffers, etc.) are freed? In a few paragraphs, describe the strategy or strategies you adopted for managing these issues. Give an example.
>
> 任何在用户指定的地址上对用户程序的内存的访问可能因为指针错误而失败。此类访问一定导致进程终止。系统调用充满了这样的访问。如一个“写”系统调用需要先从用户栈中读系统调用号，然后每一个调用的 3 个参数，然后是任意数量的用户内存。任何这些都可能造成失败。这构成一个设计错误处理的问题：如何最好地避免混淆主要错误处理的烦恼？此外，当错误被检查到，你如何保证所有的临时开出的资源（锁、缓冲区等）都被释放？用几段话来描述你处理这些问题的策略。

### SYNCHRONIZATION

> B7: The "exec" system call returns -1 if loading the new executable fails, so it cannot return before the new executable has completed loading. How does your code ensure this? How is the load success/failure status passed back to the thread that calls "exec"?
>
> B7: 如果新的可执行文件加载失败，"exec"系统调用会返回-1，所以它不能够在该新的可执行文件成功加载之前返回。你的代码是如何保证这一点的？加载成功/失败的状态是如何传递回调用"exec"的线程的？

> B8: Consider parent process P with child process C. How do you ensure proper synchronization and avoid race conditions when P calls wait(C) before C exits? After C exits? How do you ensure that all resources are freed in each case? How about when P terminates without waiting, before C exits? After C exits? Are there any special cases?
>
> B8: 考虑有父进程 P 和它的子进程 C。当 P 在 C exit 之前调用 wait(C)时，你如何确保同步以及如何避免争用的情况？你如何确保在每种情况下，所有的资源都被释放？如果 P 在 C exit 之前，没有 waiting 便终止？如果在 C exit 之后？有什么特殊情况吗？

### RATIONALE

> B9: Why did you choose to implement access to user memory from the kernel in the way that you did?
>
> B9: 为什么你使用这种方式来实现从内核对用户内存的访问？

> B10: What advantages or disadvantages can you see to your design for file descriptors?
>
> B10: 你对文件描述符的设计有什么优劣吗？

> B11: The default tid_t to pid_t mapping is the identity mapping. If you changed it, what advantages are there to your approach?
>
> B11: 默认的 tid_t 到 pid_t 的映射是 identity mapping。如果你进行了更改，那么你的方法有什么优点？