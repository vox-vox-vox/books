---
typora-copy-images-to: upload
---

# 异常

 # 进程

## 进程地址空间

每个进程有独立、私有的地址空间。

<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gwq8iz14vgj30cu0gh761.jpg" alt="image-20211124144259654" style="zoom:87%;" />

## 进程控制块(PCB)

每个进程在 **内核** 中都有一个独立的进程控制块(Process Control Block)

```shell
{
	pid  
	文件描述符表
	进程状态 运行、挂起、停止、僵尸等
	umask 
	当前工作目录
	。。。
}
```

系统中同时运行着很多进程，这些进程都是从最初只有一个进程开始一个一个复制出来的。在Shell下输入命令可以运行一个程序，是因为Shell进程在读取用户输入的命令之后会调用fork复制出一个新的Shell进程，然后新的Shell进程调用exec执行新的程序。

## 进程状态

### 进程运行时

进程有3种状态：

- 运行态 ：该时刻进程实际占用CPU
- 就绪态 ：可运行，但因为其他进程正在运行而暂时停止
- 阻塞态 ：一个进程在逻辑上不能继续运行，除非某种外部事件发生

<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gwp69v3lh3j30xq09w3zv.jpg" alt="image-20211123163916594" style="zoom:50%;" />

对上述4个转换的解读：

- **转换1** 操作系统发现进程不能继续运行下去，例如读取一个没有东西的管道，进程会进入阻塞。
- **转换2,转换3** 进程调度程序引起的。系统认为一个运行的进程已经占用太多时间CPU，进程从运行进入就绪，反之亦然。
- **转换4** 一个外部事件发生时，进程从阻塞进入就绪。

### 进程结束后

结束但没有被父进程回收的进程处于僵死状态

## 进程调度

### 定义

有多个进程处于就绪，如果 **只有一个CPU可用** ，必须选择下一个要运行的进程。完成这个选择的程序叫做进程调度程序，所涉及的算法叫做进程调度算法。

### 进程行为

根据进程上运行程序的不同种类，进程行为分为CPU密集型和IO密集型

**CPU密集型**：较长时间CPU集中使用，较少频次IO等待

**IO密集型**：较短时间CPU集中使用，较多频次IO等待

### 何时调度

- 新启动子进程，子进程进入就绪态
- 一个进程结束，CPU空闲下来，必须进入调度
- 。。。

### 调度算法

#### 轮转调度

每个进程被分配一个时间片，时间片结束后进程仍在运行则剥夺其CPU并分配给另一个进程。在时间片结束前进程阻塞或结束则CPU立即切换。

该算法维护一个可运行进程的队列，存储处于就绪状态的进程。每次时间片用完后放入队尾：

<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gwp7gfckaej31as0ckgod.jpg" alt="image-20211123172023419" style="zoom:33%;" />

时间片过短：考虑到每个进程上下文切换的耗时，时间片过短时进程执行程序有效时间很短

时间片过长：存在多个进程时，队列后面的进程被迫等待很久

时间片大概取：**40-50ms**

### 调度算法指标

吞吐量：

CPU利用率：

最小响应时间：







## 用户模式和内核模式

### 定义

linux中CPU用一个寄存器描述当前进程的模式

当设置了此模式位时，**进程运行在内核模式中** 。可以使用所有指令，访问所有内存。

没有设置此模式位时，**进程运行在用户模式中。** 可以使用部分指令，访问部分内存。

处于内核模式的进程他 **还是同一个进程** ，但是拥有更高的权限

### 切换

用户模式切换到内核模式的方法：通过中断、故障、系统调用

**系统调用**：`pid=fork()` 进程进入内核模式，新建一个子进程





## 进程上下文切换

  - 整个上下文切换的过程：1. 保存当前进程上下文 2. 恢复某个先前进程上下文 3. 将控制移交给新恢复的进程
  - 上下文切换时刻：1. **调度器决定** 在进程执行的某些时刻，内核可以决定抢占当前进程，并重新开始一个之前被抢占的进程，整个过程由内核中的调度器决定。 2. **发生异常时进入内核模式** 运行应用程序代码的进程一开始在用户模式中，发生异常时处理器进入内核模式，运行异常处理程序。返回到应用程序代码后，处理器就从内核模式改回用户模式。3. **定时中断引发** 所有系统都有某种周期性定时器中断的机制，通常为1ms或10ms，每次定时器中断内核就能判断当前进程已经运行足够长时间，并进行上下文切换。

## 环境变量

`exec`系统调用执行新程序时会把命令行参数和环境变量表传给main

<img src="https://tva1.sinaimg.cn/large/008i3skNly1gvrkhuxfg6j60i40ewq3d02.jpg" alt="image-20211025150205469" style="zoom:50%;" /> 

父进程在fork的时候，也会将环境变量复制给子进程

| 名字               | 含义                                   |
| ------------------ | -------------------------------------- |
| PATH               | 可执行文件的搜索路径                   |
| SHELL              | 当前shell，一般是`/bin/bash`           |
| HOME               | 当前用户主目录的路径，如`Users/huxiao` |
| 自定义（如SUFFIX） | 如`hhxx`，可以方便调试                 |

## 进程控制函数

### 进程创建-fork

内核根据父进程复制一个子进程，父进程和子进程的PCB相同。

<img src="https://tva1.sinaimg.cn/large/008i3skNly1gvrv2kttmmj30q40iujss.jpg" alt="image-20211025152033355" style="zoom:50%;" />

`pid = fork()` 如果是子进程，pid=0，如果是父进程，pid=子进程的pid。

父进程可以和子进程共享已经打开的文件。

### 程序执行-exec

fork创建子进程之后，子进程上执行的是和父进程相同的程序。调用exec，该进程的用户空间代码和数据会完全被新程序替换。调用exec前后进程id不发生变化

### 进程终止-kill

`kill`的本质是向进程发信号，通过信号的方式终止进程。

### 进程清除-wait/waitpid

进程终止时会关闭所有文件描述符，释放用户空间内存，但是PCB还保留着。该进程的父进程可以调用wait或waitpid对其进行清除。

如果一个进程已经终止，但是还未被wait/waitpid()清除，称为**僵死(zombie)进程**。任何进程刚终止时都是僵死进程。

如果父进程终止，而它的子进程还存在，则这些子进程的父进程改为`init`进程。init进程是一个特殊进程，pid=1，负责启动各种系统服务与清理子进程。

```c
/*wait阻塞等待第一个终止的子进程，返回清理成功的子进程pid*/
pid_t wait(int *status);
/*waitpid 可以通过pid参数指定等待哪一个子进程，同时options参数可以指定WNOHANG，取消父进程阻塞。也l's返回清理成功的子进程pid*/
pid_t waitpid(pid_t pid, int *status, int options);
```

### 获得父/子进程的pid

|      | 想获得父的                | 想获得子的                |
| ---- | ------------------------- | ------------------------- |
| 父   | `getpid`（得到自己的pid） | `pid=fork()`              |
| 子   | `getppid`                 | `getpid`（得到自己的pid） |



## 进程间通信（IPC）

进程有独立的地址空间，任何一个进程的全局变量其他的进程都看不到，所以**进程交换数据必须通过内核**

### 管道

管道（pipe）是一种最基本的IPC方式

<img src="https://tva1.sinaimg.cn/large/008i3skNly1gvrv2dby3kj315e0e2wff.jpg" alt="image-20211025155554330" style="zoom:50%;" />

创建管道：

```c
#include <unistd.h>
/*创建后，`filedes[0]`作为读端（从管道读数据的一端，可理解为标准输入），`filedes[1]`作为写端（向管道写数据的一端，可理解为标准输出）。*/
int pipe(int filedes[2]);
```

管道使用实例：

```c
  1 #include <stdlib.h>
  2 #include <stdio.h>
  3 #include <unistd.h>
  4 #define MAXLINE 80
  5 int main(void){
  6     int n;
  7     int fd[2];
  8     pid_t pid;
  9     char line[MAXLINE];
 10
 11     if(pipe(fd)<0){ //调用pipe(fd),得到fd[0]为读端，fd[1]为写端
 12         perror("pipe");
 13         exit(1);
 14     }
 15     if((pid=fork()) < 0){
 16         perror("fork");
 17         exit(1);
 18     }
 19     if(pid > 0){/*parent*/
 20         close(fd[0]);// 父进程关闭读端
 21         write(fd[1],"hello world\n",12);// 父进程向写端写入
 22         wait(NULL);
 23     }else{/*child*/
 24         close(fd[1]);// 子进程关闭写端
 25         n = read(fd[0],line,MAXLINE);// 子进程向读端读入
 26         write(STDOUT_FILENO,line,n);
 27     }
 28     return 0;
 29 }
```

管道使用限制：

- 单向性。如果想要从父进程-->子进程，**则父进程关闭读端，子进程关闭写端**
- 如果所有写端文件描述符都关闭，仍然有进程从读端读数据，在剩余数据被读完以后，再次读会返回0（就像读到文件末尾）
- 如果写端文件描述符未关闭，仍然有进程从读端读数据，在剩余数据被读完以后，再次读会阻塞，直至管道有新数据写入
- 如果所有读端文件描述符都关闭，仍有进程从写端向管道内部写入，该进程会收到信号`SIGPIPE`（通常导致进程异常终止）
- 如果读端文件描述符未关闭，仍有进程从写端向管道内部写入，管道没写满不会阻塞，写满时会阻塞。

### 其他方式

- 父进程通过fork可以将打开文件的描述符传递给子进程

- 子进程结束时，父进程调用wait可以得到子进程的终止信息

- 几个进程可以在文件系统中读写某个共享文件，也可以通过给文件加锁来实现进程间同步

- 进程之间互发信号，一般使用SIGUSR1和SIGUSR2实现用户自定义功能

- 管道

- FIFO

- mmap函数，几个进程可以映射同一内存区

- UNIX Domain Socket，目前最广泛使用的IPC机制

# 信号

## 基本信号介绍

<img src="https://tva1.sinaimg.cn/large/008i3skNly1gvrv26p7irj30zc0aomze.jpg" alt="image-20211025162537493" style="zoom:50%;" />

常用的：

- SIGCHLD 子进程终止时向父进程发送SIGCHLD信号
- SIGINT `ctrl+C 触发` 程序终止
- SIGQUIT  `ctrl+\ 触发` 同样是程序终止，收到SIGQUIT退出时会产生core文件 (Core dump)

## 信号产生

- 终端按键产生

`ctrl+C`   `ctrl+\` 

- 终端调用系统函数

`kill -SIGINT 25535`          25535是pid

- 软件产生

`alarm(10)` 设定一个闹钟，过了10s就向该进程发送`SIGCHLD`信号

## 信号阻塞

### 信号阻塞机制

信号阻塞机制如下：

<img src="https://tva1.sinaimg.cn/large/008i3skNly1gvrv20qai3j310o0cmmxv.jpg" alt="image-20211025192655795" style="zoom:50%;" />

- block 类似掩码，代表进程能够阻塞的信号集
- pending 代表正在阻塞的信号，如果信号解除阻塞，就将其置0
- handler 代表递达信号的处理方式，可以采用自定义的handler代替系统默认内容。

以上图为例，进程不阻塞SIGHUP，默认处理方式为SIF_DFL；进程SIGINT，且正在有一个信号被阻塞，如果解除阻塞，默认处理方式为SIG_IGN；进程阻塞SIGQUIT，但当前没有信号被阻塞，如果解除阻塞，处理方式为自定义的sighandler()

### 信号集操作函数

`sigset_t`是内置类型，对每一种信号用一个bit表示有效或无效（相当于掩码）。也称为**信号集**

- 下列函数可以改变信号集：

<img src="https://tva1.sinaimg.cn/large/008i3skNly1gvrv1ui9clj311807amy4.jpg" alt="image-20211025193730518" style="zoom:50%;" />

- `sigprocmask`可以读取或更改进程的信号集

<img src="https://tva1.sinaimg.cn/large/008i3skNly1gvrv1qtxllj310u03c3yo.jpg" alt="image-20211025194751321" style="zoom:50%;" />

​		`set`非空，将当前进程信号集设置为set；`oset`非空，保存当前进程信号集到oset；`set` `oset` 均非空，备份oset并设置为set

​		`how`可选：

<img src="https://tva1.sinaimg.cn/large/008i3skNly1gvrv1mmk5oj311607g75z.jpg" alt="image-20211025195558887" style="zoom:50%;" />

- `sigpending`读取当前进程的pending信号集，通过set传出

  <img src="https://tva1.sinaimg.cn/large/008i3skNly1gvrv1crzyaj311603mglo.jpg" alt="image-20211025195953961" style="zoom:50%;" />

  

  

  

## 信号捕捉

### 捕捉信号流程

捕捉流程如下：

1. 注册`sighandler()`
2. 程序在main中，信号到来，切换到内核
3. 内核决定将控制递交给注册的`sighandler()`，它采用和main函数不同的堆栈空间，是两个独立的控制流程
4. `sighandler()`结束，再次返回内核，然后返回main

<img src="https://tva1.sinaimg.cn/large/008i3skNly1gvrv14sum0j30vc0iodhl.jpg" alt="image-20211025201358459" style="zoom:50%;" />

### sigaction

`sigaction`可以读取和修改指定信号的行为，主要用来注册`sighandler()`

<img src="https://tva1.sinaimg.cn/large/008i3skNly1gvrv2xxaxrj310402aaa8.jpg" alt="image-20211025202230321" style="zoom:50%;" />

与`sigprocmask`相似，`act`非空，将当前进程信号行为设定为act；`oact`非空，保存当前进程信号行为设定为oact；`oact` `act` 均非空，备份oact并设置为act

其中，sigaction是一个抽象结构体，设定了进程对信号的处理行为：

```c
struct sigaction{
     void        (*sa_handler)(int);           // signal handler 地址 or SIG_IGN(忽略信号) or SIG_DFL(执行系统默认)
     sigset_t    sa_mask;                      // additional signals to block
     int         sa_flags;                     // signal options
     void        (*sa_sigaction)(int,siginfo_t *,void *);    //alternate handler
}
```

### pause

`pause`使进程挂起，直至有信号到达

## 信号带来的并发问题

由于信号处理函数`sighandler()`实际上和程序运行的进程属于两个互相独立的流程(**属于相同的进程🐴**)。如果**多个控制流程操作共享资源**，就会出现问题

### 可重入

操控共享资源的函数不应设为可重入，否则会出现并发问题：

<img src="https://tva1.sinaimg.cn/large/008i3skNly1gvrv0xuj9hj316g0q4jtg.jpg" alt="image-20211025204130889" style="zoom:50%;" />

### sig_atomic_t与volatile



### 竞态条件与sigsuspend



# 线程

## 线程控制

> 以下线程均为调用POSIX标准线程库情况下的线程

### 线程id

`pthread_t`类型

每个进程的pid在系统中唯一

每个线程的tid在进程中唯一，系统中不唯一

### 创建线程

```c
pthread_create
int pthread_create(&线程id, 线程属性(一般写NULL) , 函数指针,传入函数的参数);  
```

### 终止线程

- 任意线程调用exit，整个进程终止
- 线程函数return(从main函数return相当于调用exit，导致整个进程终止)
- 线程可以通过pthread_exit终止自己

## 线程同步与其并发问题

### mutex

### condition varible

### semaphore

### 其他同步方式



# 线程和进程

### linux线程和进程的关系

- linux中，进程通过`task_struct`表示，不像其他操作系统会区分进程，轻量级线程和线程，linux系统用`task_struct`来表示所有的上下文。一个单线程进程有一个`task_struct`，一个多线程进程每个用户级线程都有一个`task_struct`
- `task_struct `<==> [PCB](## 进程控制块(PCB))
- Linux中，进程是资源分配单位，线程是调度单位
- 实际上执行代码的单位是线程，所以线程才是CPU的调度单位，看起来像是进程的原因是单线程进程
- CPU进行调度的单元是PCB
- ![image-20211124134437110](https://tva1.sinaimg.cn/large/008i3skNgy1gwq6u8hl5zj316g0jgabw.jpg)

### fork和pthread_create

fork和pthread_create本质上都调用了底层的clone

#### fork

fork调用时复制进程的地址空间和PCB，相当于创建新的进程。

系统调用fork时，陷入内核并创建一个task_struct和其他相关数据结构（如内核堆栈，thread_info等）。task_struct的值主要根据父进程的内容进行填充，并且寻找一个新的PID写进去，并将<PID,task_struct>存入hash表

理论上，接下来应该复制子进程的数据段和堆栈，但是由于复制太过昂贵，linux实际采用了一种 **写时复制** 的方法：将子进程中的内容指向父进程（而不是复制）并将父进程内容设为只读，如果需要写入时才会为该页面分配一个新的副本：

<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gwqd9oidgzj31ry0lijtu.jpg" alt="image-20211124172703622" style="zoom:67%;" />

#### pthread_create

pthread_create调用时复制PCB但不复制地址空间，创建的新线程和原线程公用进程的地址空间













































