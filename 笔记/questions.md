# questions

## IO

- 阻塞

读写常规文件不会发生阻塞，阻塞发生在读写终端和网络中
读终端时没有换行符，会发生阻塞
读网络请求，如果对面没有包过来，也会发生阻塞

文件描述符（int）和FILE*
进程描述符=task_struct=PCB(process control block)
文件描述符表=file_struct,其中每一个表项包含一个指向已打开文件的指针。
文件描述符表=指针数组；文件描述符=指针数组的index
对于C标准IO库来说，打开的文件由FILE*标识；对内核来说打开的文件由文件描述符标识

- 
  1. 内核和磁盘不在一起嘛
  2. 什么叫传给内核，内核再写回设备，这居然是两个过程
  3. 用户空间/内核空间，用户程序/内核，C标准IO缓冲区/磁盘 之间的关系
  4. 标准输入输出 和 终端设备有什么关系？ 
- 用户调用read，内核调用的read和用户调用的read居然不是一个read吗
- IO多路复用

## socket

- 为什么网络通信不要用标准IO

- ~~什么是接收缓冲区，还有发送缓冲区吗?~~

- 窗口就是接收/发送缓冲区吗，对于全双工通信每一端的窗口是不是有两套？一个发送窗口一个接收窗口？

- TCP面向流，这里的流到底应该怎么理解？和普通面向消息的UDP有什么本质的区别

- TCP 中RST是reset。如果server端收到但是无法找到对应连接，就返回一个带RST的报文给cli. 然后cli端会发生什么？在TCP层会有什么行为吗？为什么再次调用write的时候会发送SIGPIPE？TCP层的一些默认操作时由谁来触发或者完成的？

- socket listenfd和connfd有什么区别？

- ip地址0.0.0.0和127.0.0.1和192.168.xxx有什么区别

- 为什么查看端口8000有3行出现，都说明了什么意思

- socket与TCP对应：
    1. close 对应 发送FIN，主动关闭单向连接。server关闭时，也会向cli发送FIN

  2. read EOF对应收到了对面发来的FIN
  3. connect()代表发送SYN
  4. accept()代表回复一个SYN-ACK段
  5. Listen 代表server状态变为LISTEN

- socket 和 open到底有什么不一样？打开以后在内核会有什么地方显示它是一个socket文件吗

## process

- 进程状态
  sleep：阻塞
  running：

  1. 正在被调度执行。CPU处于该进程的上下文环境中，程序计数器（eip）里保存着该进程
  的指令地址，通用寄存器里保存着该进程运算过程的中间结果，正在执行该进程的指令，
  正在读写该进程的地址空间。

  2. 就绪状态。该进程不需要等待什么事件发生，随时都可以执行，但CPU暂时还在执行另一
     个进程，所以该进程在一个就绪队列中等待被内核调度。系统中可能同时有多个就绪的进
     程，那么该调度谁执行呢？内核的调度算法是基于优先级和时间片的，而且会根据每个进
     程的运行情况动态调整它的优先级和时间片，让每个进程都能比较公平地得到机会执行

​		挂起

​		僵尸

​		。。。

## 网络

驱动到底是啥，工作在链路层？执行ARP协议的是驱动吗？执行IP协议的是驱动吗

网卡是啥，工作在哪一层？

以太网是局域网吗，为什么能做为最底层的，按理说最底层不应该是最广泛的吗

既然网络传输只有IP地址，那封装到帧里面怎么看到IP地址的啊

**地址！！**硬件地址，IP地址，IP地址＋端口，域名，

将套接字接口函数实现为系统调用，这些系统调用会陷入内核，可以理解成暴露函数接口给用户态的内核函数？

接口=网卡？

## 突然想到...

- 并发编程的一些思想：race-condition（竞态条件），if-condition之类的（先判断再操作）
- 加锁带来上下文切换的开销
- 虚拟地址空间
- CAS轮询期间可以干什么，如何避免CPU空转
- 单位均为字节。一字节=8bit=2位16进制数
- 对称加密和非对称加密
- Poll：非阻塞轮询 select：阻塞多路并行
- golang中文件以流的形式，怎样理解流
- 4次挥手的善后到底是干嘛的
- 服务器出现大量close_wait怎么处理
- 两个线程之间的“同步”到底指什么
- /etc/hosts 表示不进行DNS而是直接从这里面找

# 11.9

看完了一遍不知道从何入手，感觉对计网和OS有了一定认知但是还不够深刻，决定今天一天整理一下

之前自己学过的主要有：process/IO/socket(TCP) 先就对这三个方面做一定的整理，不会的或者不懂的都记录在这里

脱离了实际的理论是空洞的，抽象的，不堪一击的。实践是检验真理的唯一标准



# 11.29

为什么用红黑树不用AVL 红黑树插入效率更快
