# Project 2 实验文档

[TOC]



## 团队 那遺矢dé圊舂|ゞ

### 基本信息

| 姓名   | 学号     | Git 用户名   |
| ------ | -------- | ------------ |
| 黄森辉 | 19231153 | BUAAhuangsh  |
| 陈逸然 | 19231005 | aurora-cccyr |
| 吴浩华 | 19231023 | async222     |
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

#### 准备工作

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

为了保证文件访问参数的有效性，我们在syscall_handler中检测第一个参数的合法性。

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

**`sys_WRITE`**

`sys_WRITE`可以实现系统向文件或屏幕写数据，具体实现思路是从用户栈中读取三个参数：fd，buffer和size。fd为写入类型，如果是向屏幕写入，则直接在终端输出，否则找到文件标识符并写入文件。（打开文件表将在打开文件时建立，具体参见sys_open，file_write函数由pintos提供）

```c
void 
sys_write (struct intr_frame* f)
{
  uint32_t *user_ptr = f->esp;
  check_ptr2 (user_ptr + 7);
  check_ptr2 (*(user_ptr + 6));
  *user_ptr++;
  int temp2 = *user_ptr;
  const char * buffer = (const char *)*(user_ptr+1);
  off_t size = *(user_ptr+2);
  if (temp2 == 1) {
    /* 写入标准输出 */
    putbuf(buffer,size);
    f->eax = size;
  }
  else
  {
    /*写入到文件 */
    struct thread_file * thread_file_temp = find_file_id (*user_ptr);
    if (thread_file_temp)
    {
      acquire_lock_f ();
      f->eax = file_write (thread_file_temp->file, buffer, size);
      release_lock_f ();
    } 
    else
    {
      f->eax = 0;
    }
  }
}
```



#### 系统调用_进程

进程方面的系统调用主要涉及`lib/syscall­nr.h`中前四个服务，对应的我们需要写四个函数：

```c
void sys_halt(struct intr_frame* index);
void sys_exit(struct intr_frame* index);
void sys_exec(struct intr_frame* index); 
void sys_wait(struct intr_frame* index);
```

##### `sys_halt`

`sys_halt`是关机函数，通过运行`devices/shutdown.h`中声明的`shutdown_power_off()`函数终止Pintos；

```c
// devices/shutdown.h

/* Powers down the machine we're running on,
   as long as we're running on Bochs or QEMU. */
void
shutdown_power_off (void)
{
  const char s[] = "Shutdown";
  // ...
}
```

##### `sys_exit`

`sys_exit`是退出进程函数，在`thread.c`中新增退出码`st_exit`以供后续使用；

```c
thread_current()->st_exit=*(((int*)f->esp)+1);//为当前线程保存退出码
thread_exit();
```

其中`f->esp`指向调用函数名，后+1代表系统调用的第一个参数，如下函数同理；

另外，该函数与后两种函数，都需调用`check_ptr2()`以检验参数正确性，特别的，对于函数`sys_exec()`还需检验存放调用函数所需要的参数地址的正确性，因此对于这两种调用定义函数如下：

```c
uint32_t
zxyA(struct intr_frame* f)
{
  uint32_t *ptr = f->esp;
  check_ptr2 (ptr + 1);
  *ptr++;
  return *ptr;
}

uint32_t
zxyB(struct intr_frame* f)
{
  uint32_t *ptr = f->esp;
  check_ptr2 (ptr + 1);
  check_ptr2 (*(ptr + 1));
  *ptr++;
  return *ptr;
}
```

##### `sys_exec`

`sys_exec`是开始另一程序的函数，调用了参数传递拟写的`process_execute`函数；

```c
f->eax = process_execute((char*)(f->esp));
```

##### `sys_wait`

`sys_wait`是等待子进程函数，根据需求，我们还需要在`process.c`文件中补写`process_wait`函数；

```c
f->eax = process_wait(((int*)f->esp)+1);
```

`process_wsit`函数原先已被写好，留待我们补充：

```c
/* Waits for thread TID to die and returns its exit status.  If
   it was terminated by the kernel (i.e. killed due to an
   exception), returns -1.  If TID is invalid or if it was not a
   child of the calling process, or if process_wait() has already
   been successfully called for the given TID, returns -1
   immediately, without waiting.

   This function will be implemented in problem 2-2.  For now, it
   does nothing. */
int
process_wait (tid_t child_tid UNUSED) 
{
  return -1;
}
```

通过注释我们了解到，这个函数在系统调用部分的作用为等待线程TID结束并返回其退出状态（现在始终返回-1）；如果因异常终止，则返回-1；如果TID无效或不是当前进程的子进程或已调用`process_wait()`，则返回-1.

在编写代码之前，我们需要统筹在实现`wait`的过程中都需要父子进程的哪些属性；因为需要知道父进程有哪些子进程，所以需要一个属性装有父进程拥有的子进程，对于返回值还需要增加返回状态；对于被装的子进程，我们也需要单独增加结构体子进程来标识其属性；最后在创建进程时，还需要初始化子进程以实现程序的正常运行；

为了实现如上情况，我们在`thread.h`中添加了新的属性：

```c
struct child
  {
    bool isrun;//运行状态
    struct list_elem child_elem;//子进程列表
    struct semaphore sema;//控制父进程等待的信号量
  };
```

同时修改了`thread.c/thread_create()`函数对子进程进行初始化；

```c
  t->thread_child = malloc(sizeof(struct child));
  t->thread_child->tid = tid;
  sema_init (&t->thread_child->sema, 0);
  list_push_back (&thread_current()->childs, &t->thread_child->child_elem);
  t->thread_child->store_exit = UINT32_MAX;
  t->thread_child->isrun = false;
```

实现代码如下：

首先找到当前子进程序列：

```c
struct list *childList = &thread_current()->childs;
```

为遍历该序列定义起始量`indexA`和当前量`indexB`：

```c
struct list_elem *indexA;
struct child *indexB;
```

在`indexA!=list_end(childList)`时循环遍历该子进程序列，如果当前判断的进程是子进程，则其停止运行，减少信号量以唤醒父进程，否则返回-1，否则`indexA`“++”；

```c
void
modifyChild(struct child *c)
{
  c->isrun = true;
  sema_down (&c->sema);
}
```

```c
int
judgeChild(tid_t child_tid,struct child *c)
{
  if (c->tid == child_tid){
      if (!c->isrun){modifyChild(c);return 1;} else return -1;
    }
  return 0;
}
```

如果循环全部也没能找到子进程，则直接返回-1；如果中途尚未返回-1，执行至此则说明判断成功，删除子进程以给父进程空出位置，最后返回；

```c
list_remove (indexA);
return indexB->store_exit;
```

#### 系统调用_文件

系统调用中的文件部分，有 create, remove, read, open, filesize, seek, tell, and close这几个函数。

```c
void sys_create(struct intr_frame* f); /* syscall create */
void sys_remove(struct intr_frame* f); /* syscall remove */
void sys_open(struct intr_frame* f);/* syscall open */
void sys_filesize(struct intr_frame* f);/* syscall filesize */
void sys_read(struct intr_frame* f);  /* syscall read */
void sys_seek(struct intr_frame* f); /* syscall seek */
void sys_tell(struct intr_frame* f); /* syscall tell */
void sys_close(struct intr_frame* f); /* syscall close */
```

`create`

创建一个文件，首先获取命令行参数，包括了文件名，地址。check保证合法性，然后获取文件的锁，调用`filesys_create`创建文件，最后需要释放锁。

```c
bool filesys_create (const char *name, off_t initial_size) 
{
 block_sector_t inode_sector = 0;
 struct dir *dir = dir_open_root ();
 bool success = (dir != NULL
​         && free_map_allocate (1, &inode_sector)
​         && inode_create (inode_sector, initial_size)
​         && dir_add (dir, name, inode_sector));
 if (!success && inode_sector != 0) 
  free_map_release (inode_sector, 1);
 dir_close (dir);
 return success;
}
```

`remove`
删除一个文件，check保证命令行传来的参数合法性，获取锁，调用`filesys_remove`删除，释放锁.

```c
bool filesys_remove (const char *name) 
{
 struct dir *dir = dir_open_root ();
 bool success = dir != NULL && dir_remove (dir, name);
 dir_close (dir); 
 return success;
}
```

`open`
打开一个文件，check保证命令行传来的参数合法性，获取锁，调用`filesys_open`打开文件，释放锁。获取当前线程，并且为文件添加一个属于当前线程的文件标识符，放入线程的打开文件的列表中，返回文件的标识符。

```c
struct file * filesys_open (const char *name)
{
 struct dir *dir = dir_open_root ();
 struct inode *inode = NULL;
 if (dir != NULL)
  dir_lookup (dir, name, &inode);
 dir_close (dir);
 return file_open (inode);
}
```

`filesize`
通过`find_file_id`获取文件标识符。调用`file_length`返回以文件标识符fd指代的文件的大小。

```c
struct thread_file * find_file_id (int file_id)
{
 struct list_elem *e;
 struct thread_file * thread_file_temp = NULL;
 struct list *files = &thread_current ()->files;
 for (e = list_begin (files); e != list_end (files); e = list_next (e)){
  thread_file_temp = list_entry (e, struct thread_file, file_elem);
  if (file_id == thread_file_temp->fd)
   return thread_file_temp;
 }
 return false;
}
```

`seek`
通过`find_file_id`获取文件标识符，获取文件的锁，根据传入的参数，使用`file_seek`函数把下一个要读入或写入的字节跳转到指定文件的指定位置。

`tell`
通过`find_file_id`获取文件标识符，获取文件的锁，根据传入的参数，使用`file_tell`函数把下一个要读入或写入的字节的位置返回。

`close`
通过`find_file_id`获取文件标识符，获取文件的锁。如果文件打开，调用`file_close`关闭文件，关闭后，把这个文件从线程的文件list中移除并释放资源。

```c
void sys_close (struct intr_frame* f)
{
 uint32_t *user_ptr = f->esp;
 check_ptr2 (user_ptr + 1);
 *user_ptr++;
 struct thread_file * opened_file = find_file_id (*user_ptr);
 if (opened_file)
 {
  acquire_lock_f ();
  file_close (opened_file->file);
  release_lock_f ();
  /* Remove the opened file from the list */
  list_remove (&opened_file->file_elem);
  /* Free opened files */
  free (opened_file);
 }
}
```



## 重难点讲解

测试样例中提供了这样一个测试 multi­oom,该测试通过用户进程不断地做递归调用，并且不断地进行打开文件操作，并随机地终止某一个线程。该测试考察到了系统平均负载，资源利用率，以及系统的稳定性，具体考察过程如图：

![image-20211208113854841](C:/Users/aurora/AppData/Roaming/Typora/typora-user-images/image-20211208113854841.png)

**忙等待造成超时处理**

在之前设计 process_wait 函数中，我们设置了一个 while(1)死循环，利用get_thread_ by_tid (child_tid)函数得到目标进程，然后检查该进程的运行状态,发现状 态是 THREAD_DYIN G，　即可得到返回值退出循环。 

但是在实际运行过程中，随着递归层数的加深，进程创建的速度越来越慢，测试程序有一个时间限制：360 秒，然而采取刚才所说的方法不能按时跑完。我在调试过程中，注意检查创建线程所调用的函数，发现没有任何操作的耗时会随着线程数量的增加而增加，加入调试信息发现，创建线程的每一步操作耗时都是线性增长的，由此判断出程序调度过程中发生了忙等待。此时，我们设计出利用信号量的方法实现 process_wait()函数，解决了测试超时的问题。当程序递归调用时，process_wait()在子线程运行完毕之前会通过 sema_down()进入子线程 wsem 成员变量的 wait_list 里，在子线程运行完毕进入 process_exit()阶段sema_up()该信号量唤醒父线程。这样一来，在递归调用过程中，实际上只有一个线程在ready_list 之中，递归速度大大加快。

**内存资源释放操作**

在 multi­oom 运行时，进程会不停地打开文件，并且不去关闭，如果不在进程退出时关闭这些文件，释放这些文件的文件描述符(fd)，会造成内存泄漏，影响系统效率。

解决办法首先是修改 struct thread 数据结构，向前文描述的，加入 fd_list,用来保存由该进程打开文件的文件描述符，在进程退出函数 process_exit()之中，遍历 fd_list 的每一个结点，通过 list_entry 函数解析出文件指针，用 file_close()函数来关闭所有文件，之后，调用 free 函数释放文件描述符 fd 的内存空间，这样便可以防止内存资源的泄漏，在这个方面上保证pintos 操作系统的性能。

**系统平均负载能力测试**

系统平均负载能力反映在同时处于就绪状态的线程数，在本测试中，对应的就是递归调用的层数，层数越多，并发能力越强，而就绪程序越多，对于系统的压力就越大。我们需要测试出 pintos 操作系统的并发能力上限，对用户创建线程做出一定限制，以保证 pintos 系统正常运行。

### `sys_exec`流程图

![avatar](sys_exec.png)

### `sys_wait`流程图

![avatar](sys_wait.png)

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

### 赵晰莹

这次实验中我负责的是系统调用的进程部分，实则与各个部分都有衔接，联系紧密。`syscall.c`中大多是规范操作，取参数、检查参数及其它需要的量、调用其它函数以解决，整体而言算是承上启下的作用；`process.c/process_wait`中涉及到了一些父子进程知识，总体上还是一个判断子进程的大循环，我们所使用的信号量使能合理有效地运行；另外我还修补了`thread.c`和`thread.h`，大约是边边角角的增加属性与初始化方法。

起初的分工是一个难点，我们三个人搞系统调用无法明确界限，而我以为还是明确的分工才能得到高效的成果。于是在我们三个人的反复商量后，剖析出了`syscall.c`的一堆需要填补的函数，终于可以大约明确的分个工了。p2不比p1，没那么明确的承接，流水线作业优点是合并容易，但是缺点也是一堆bug和每两个人的思路都小有差异，第一次写完之后一个点都过不了真的是非常崩溃......后来我们又聚在一起研讨了好久，一个函数一个函数的讨论，才终于能过了80个点，尤其是一个细小的差异都能从3个点不过到50个点不过（亲身经历T T）。后来又经过添加详细注释与多次优化代码终于有了最后的成品，想想从开始啥也不会到现在也能胡诌一个操作系统，还是在时间的淬炼里有了不知不觉的提升。

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

- <thread.c> 

  - 建立了一个文件锁.

    ```C
    /*使用文件锁来保证文件操作时的安全性*/
    static struct lock lock_file;
    ```

  - 上锁和放锁函数。

    ```c
    /*获取线程的文件锁*/
    void acquire_lock_file (){
     lock_acquire(&lock_file);
    }
    /*释放线程的文件锁*/
    void release_lock_file (){
     lock_acquire(&lock_file);
    }
    ```



- <thread.h> 

  - 

    ```c
    struct thread_file {
      int fd;
      struct file* file;
      struct list_elem file_elem;
     };
    ```

  - struct thread中新增

    ```C
    struct list files;        /* List of opened files */
    int file_fd;             /* File's descriptor */
    struct file * file_owned;     /* The file opened */
    ```

    



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

**详细可查看“需求分析-系统调用-系统调用_进程中`sys_wait`函数与重难点分析-`sys-wait`；**

简单而言，遍历所有子进程，对其进行判断，如果是当前进程的子进程还已经运行结束，就退出，返回退出状态；如果是当前进程的子进程但是没运行结束，就接着等待；如果遍历结束也没找到则返回-1。

> B6: Any access to user program memory at a user-specified address can fail due to a bad pointer value. Such accesses must cause the process to be terminated. System calls are fraught with such accesses, e.g. a "write" system call requires reading the system call number from the user stack, then each of the call's three arguments, then an arbitrary amount of user memory, and any of these can fail at any point. This poses a design and error-handling problem: how do you best avoid obscuring the primary function of code in a morass of error-handling? Furthermore, when an error is detected, how do you ensure that all temporarily allocated resources (locks, buffers, etc.) are freed? In a few paragraphs, describe the strategy or strategies you adopted for managing these issues. Give an example.
>
> 任何在用户指定的地址上对用户程序的内存的访问可能因为指针错误而失败。此类访问一定导致进程终止。系统调用充满了这样的访问。如一个“写”系统调用需要先从用户栈中读系统调用号，然后每一个调用的 3 个参数，然后是任意数量的用户内存。任何这些都可能造成失败。这构成一个设计错误处理的问题：如何最好地避免混淆主要错误处理的烦恼？此外，当错误被检查到，你如何保证所有的临时开出的资源（锁、缓冲区等）都被释放？用几段话来描述你处理这些问题的策略。

### SYNCHRONIZATION

> B7: The "exec" system call returns -1 if loading the new executable fails, so it cannot return before the new executable has completed loading. How does your code ensure this? How is the load success/failure status passed back to the thread that calls "exec"?
>
> B7: 如果新的可执行文件加载失败，"exec"系统调用会返回-1，所以它不能够在该新的可执行文件成功加载之前返回。你的代码是如何保证这一点的？加载成功/失败的状态是如何传递回调用"exec"的线程的？

**详细可查看重难点分析-`sys_exec`；**

我们使用了信号量同步机制的exec流程，start_process后会进行sema_down，直到成功加载后才会sema_up；

通过返回确定的tid（>0成功，-1失败）传递给父进程。

> B8: Consider parent process P with child process C. How do you ensure proper synchronization and avoid race conditions when P calls wait(C) before C exits? After C exits? How do you ensure that all resources are freed in each case? How about when P terminates without waiting, before C exits? After C exits? Are there any special cases?
>
> B8: 考虑有父进程 P 和它的子进程 C。当 P 在 C exit 之前调用 wait(C)时，你如何确保同步以及如何避免争用的情况？你如何确保在每种情况下，所有的资源都被释放？如果 P 在 C exit 之前，没有 waiting 便终止？如果在 C exit 之后？有什么特殊情况吗？

与上题类似，`wait(C)`先sema_down信号量阻塞父进程，而后sema_up唤醒父进程；

在当前进程退出时只释放子进程的资源，保留自己thread结构直到父进程退出再释放；

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