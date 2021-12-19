---
typora-copy-images-to: upload
---

# socket

- socket编程属于**应用层**

- **套接字** 
  
  - 从linux内核的角度看，一个套接字是**连接的一个端点**，
  - 从linux程序的角度看，套接字是一个**有相应描述符的打开的文件**
  
- **套接字地址**
  
  - 从linux内核的角度看，每个套接字都有相应的**套接字地址，格式为{ 地址 : 端口 }**。客户端发起一个连接请求时，套接字地址的端口是由内核自动分配的，服务端套接字地址的端口通常是固定的(如80)
  - 从linux程序的角度看，套接字地址为[sockaddr](#socket地址)
  
- **套接字描述符**

  linux程序中打开的网络文件所对应的描述符就是套接字描述符，即：`套接字描述符<=>网络文件描述符<=>sockfd`。**有的时候，套接字描述符和套接字不予以区分，都表示sockfd**

- **套接字接口** 

  套接字接口是一组函数，它们和Unix I/O函数结合起来，用于创建网络应用。

- **TCP连接**

  一条TCP连接可以用 **{服务端地址:服务端端口:客户端地址:客户端端口}** 唯一标识，同一IP同一端口的server可以和多个client建立TCP连接

  如果给定serverAddr,serverPort,clientAddr,clientPort，一次TCP握手建立两条TCP连接，分别是`{serverAddr,serverPort,clientAddr,clientPort}`和`{clientAddr,clientPort,serverAddr,serverPort}`

- 各个版本对套接字描述符的称呼各不相同，有叫“监听描述符”的，有叫“监听套接字”的... 为了予以区分，统一用`listenfd`和`connfd`来称呼

# socket地址

socketAPI是一层抽象的网络编程接口，适用于各种底层协议，如IPV4、IPV6、UNIX Domain Socket等，但是各种网络协议的地址不相同:

- IPV4:IPV4地址用 sockaddr_in 表示，包括16位端口号和32位IP地址
- IPV6:IPV6地址用 sockaddr_in6 表示，包括16端口号、128IP地址和一些控制字段
- UNIX Domain Socket:用sockaddr_un结构体表示

虽然地址不同，但也有一部分相同。各种socket地址结构体的开头是相同的，前16位是长度，后16位表示整个地址类型。    

<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gwchkv9betj30jz0dkzl0.jpg" alt="image-20211112171818314" style="zoom:80%;" />

这样，为了编程的通用性，socket地址统一都用 struct sockaddr* 类型表示，只需要取得sockaddr结构体中的首地址，可以根据地址类型字段确定sockaddr中的内容。

# 基于tcp协议的网络程序

## TCP在linux内核中的实现

TCP/IP协议的实现在内核完成，下图是一种抽象模型

![image-20211113133248065](https://tva1.sinaimg.cn/large/008i3skNgy1gwdgokzhm6j30xe0ak0uf.jpg)



- TCP协议栈在每台linux内核中维护两个缓冲区，一个是发送缓冲区，一个是接受缓冲区。所有对套接字(从linux程序的角度看，套接字是一个**有相应描述符的打开的文件**)的读取/写入操作实际上都是向缓冲区里读取/写入

  socket缓冲区大小可以通过`setsockopt`函数设置

- 从缓冲区到NIC(网卡)的数据转移过程由DMA完成，不必在意细节。网卡的发送数据过程也不必在意细节。

## TCP连接的具体过程

### socket API层面示意图

<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gwdt33xnb4j30og114jt2.jpg" alt="image-20211113204154528" style="zoom:60%;" />

### socket如何与TCP一一对应

![image-20211113171959056](https://tva1.sinaimg.cn/large/008i3skNgy1gwdn8xp8q5j30p50nfgng.jpg)

- `listen`后，主动套接字变为被动套接字，server端由CLOSE进入LISTEN状态，然后阻塞在`accept`

- `connect`后，client端发送SYN的报文，server端内核中socket buffer收到SYN，在内核中进行三次握手，并向连接已完成队列中添加；

- `accept`时，阻塞消费“连接已完成队列”中的信息，消费成功后生成新的connfd。需要注意的是，三次握手建立连接时服务端对应的套接字是listenfd，即建立的TCP连接为 **{clientIP:clientPort:serverIP:serverPort}** 而调用`accept`后，这个四元组并没有改变，说明TCP连接还是原来那个连接，只不过服务端对应的套接字变为connfd，这样listenfd才可以继续用来监听

  <img src="https://img2018.cnblogs.com/blog/733013/201903/733013-20190308141545430-875305852.png" alt="img" style="zoom:67%;" />

- client端`close`时，client发送FIN报文给server，同时server的内核返回ACK给client端。在server的用户态来看，FIN报文收到后体现为`read`返回0，可以自己安排善后工作，这里对应着4次挥手的前两次已经结束。

  ![image-20211113161949142](https://tva1.sinaimg.cn/large/008i3skNgy1gwdliblvejj30fb010t8m.jpg)

  由上图可以看到，8022(server)进入CLOSE_WAIT，而62272(client)进入FIN_WAIT_2，证实了我们的说法，4次挥手在socket编程中不是同时发生的。

- server端进行完善后工作，然后调用`close`，发送FIN报文给client端，完成4次挥手，然后client端进入TIME-WAIT

  ![image-20211113172332866](https://tva1.sinaimg.cn/large/008i3skNgy1gwdncn3d61j30f300jglf.jpg)

- 总结：
  - 握手挥手等发送SYN，FIN报文的操作都在内核中进行

### server为什么需要listenfd和connfd

listenfd作为连接的起点，通常被创建一次，并存在于整个服务的生命周期。connfd则只存在于服务器为一个客户端服务的过程中。

只有使用listenfd和connfd才可以建立并发服务器，同时处理多个客户端的连接

## socket API

### server端&client端公有API

socket 本质就是将网络读写变成IO。公有操作实际上就是文件的读写操作：

#### socket

```c
socket     	int socket(int family,int type,int protocol);
         		input: family 一般取AF_INET; type 一般取SOCK_STREAM; protocol 一般取0  
         		output: 成功返回一个套接字描述符
         		打开一个网络通讯端口。成功的话，就像open()一样返回一个文件描述符
```

- 进程调用socket函数指定通信协议类型，并返回套接字

- 默认情况下，socket创建的套接字是主动套接字，即对应client端，将其变为server端能用的被动套接字需要调用`listen`函数。

Todo:socket 和 open到底有什么不一样？打开以后在内核会有什么地方显示它是一个socket文件吗

#### read/write

为什么socket编程中不用标准IO？

标准I/O能在同一个流上执行输入和输出，但是协同操作需要用到`fseek`,`rewind`等函数指示文件指针，网络编程中无法使用文件指针。

#### close

```c
close:			int close(int fd);
						input:	fd:要关闭的套接字
```

### server端

#### bind 

```c
bind       	int bind(int sockfd, const struct sockaddr *myaddr, socklen_t addrlen);
         		input:sockfd 套接字描述符；myaddr 套接字地址
         		output:成功返回0，失败返回-1
         		将参数sockfd和myaddr绑定在一起，使sockfd这个套接字描述符监听myaddr所描述的地址和端口号
```

- client端不用`bind`是因为client每次地址都是随机的，但是server端地址不能随机。
- 套接字地址实际上在编程中也是由{IP:端口号}唯一确定

#### listen

```c
listen     	int listen(int sockfd, int backlog);
         		input:sockfd 套接字描述符；允许有backlog个客户端处于连接状态，收到更多的连接请求则忽略
         		output:成功返回0，失败返回-1
```

`listen`函数做两件事情：

1. 将`socket`函数创建的**主动套接字变为被动套接字**，指示内核应该 **接受(Receive)** 指向该套接字的连接请求（内核默认这个套接字对应的是客户端而不是服务器，接下来三次握手时服务器的行为和客户端有很大差别，所以需要告诉内核这个套接字是服务器）。调用`listen`导致TCP的状态从CLOSED状态变为LISTEN状态
2. 参数backlog规定了内核应当为相应套接字排队的最大连接个数

#### accept

```c
accept     	int accept(int sockfd, struct sockaddr *cliaddr, socklen_t *addrlen);
         		input: sockfd 网络文件描述符；cliaddr 传出参数,返回客户端地址和端口号；addrlen 传入传出参数,传入cliaddr长度,传出
            			 客户端地址结构体实际长度，传NULL说明不关系客户端地址
         		output: 网络文件描述符，相当于一个可以读写的文件
         		服务器调用accept()接受连接。如果此时还没有客户端的连接请求，就阻塞直至有客户端连接上来。
```

如果“已完成连接队列”为空，则阻塞；否则从“已完成连接队列”中拿出一个连接并返回由内核生成的一个全新的描述符connfd

### client端

#### connect

```c
connect    	int connect(int sockfd, const struct sockaddr *seraddr, socklen_t addrlen);
         		input:sockfd 套接字描述符；seraddr server的地址,sockaddr类型,不同协议各不相同
         		output:成功返回0，失败返回-1
         		将参数sockfd去连接server的地址，使sockfd这个网络文件描述符监听seraddr所描述的地址和端口号。connect和bind的参数一
            致，bind的sockaddr参数是自己的地址，connect的sockaddr是server的地址。
```

`connect` 发起一次tcp连接，如果server还没有listen，阻塞。



## 提升程序并发能力

### BIO

传统BIO方式，采用多进程：fork+处理SIGCHLD。每一个client建立一个tcp连接

```c
// 伪代码
listenfd = Socket();
Bind(listenfd,addr);
Listen(listenfd);
while(1){
  connfd = Accept(listenfd);
  n=fork();
  if(n==0){// 子进程
    Close(listenfd);
    while(1){
      n=Read();
      if n==0{
        break // EOF
      }
      // process request
      Write();
    }
    Close(connfd);
    exit();
  }else{// 父进程
    Close(connfd);
  }
}
```



### NIO

NIO方式，采用的是单进程下的IO多路复用，可利用select/poll/epoll来完成

#### “就绪”

接下来的讨论都基于这个“就绪”的概念。它可以大概地表述为：“tcp发送到connfd中，connfd有数据了，就说明这个connfd就绪了”。但是这样的表述显然不够准确。

实际上，网卡中的数据达到内核的socket缓冲区，就可以认为connfd就绪。这里我们用一个动图表示：

![图片](https://tva1.sinaimg.cn/large/008i3skNgy1gwuwy4zmosg30iv0f7kjm.gif)

真正准确的理解，请看这篇博文：

https://murphypei.github.io/blog/2019/08/socket-ready

#### 用户模拟单线程多路IO复用

##### 如何进行模拟

首先要明白，阻塞read不可能在单线程实现多路IO复用。如果read connfd1阻塞线程，read connfd2则根本不可能运行到，也无法IO复用。所以，要想多路复用，首先必须取消系统调用的阻塞，不然不可能实现多路同时读写：

```c
// 基于fcntl，取消read的阻塞，每次未就绪则返回-1
fcntl(connfd, F_SETFL, O_NONBLOCK);// 取消connfd 的read阻塞
int n = read(connfd, buffer) != SUCCESS);
```

![图片](https://tva1.sinaimg.cn/large/008i3skNgy1gwuwf61y92j30ll0fm74x.jpg)

这样read每次非阻塞，如果多个connfd同时并发read，也不会出现因为一个connfd阻塞而卡住其他connfd的情况，有了多路复用的雏形。

按照非阻塞的read，怎样实现对多个connfd的同时读呢？可以采用fdList维护一个已连接的connfd列表，CPU对其中的所有connfd进行遍历read，但不阻塞，如果没有就绪就返回-1，然后继续循环调用，直至不为-1：

![图片](https://tva1.sinaimg.cn/large/008i3skNgy1gwuwf2ynuxg30e10630t6.gif)

如果有就绪的connfd，就进行读取。

##### 存在的问题

- 多次对fdList进行read系统调用，十分浪费资源。

#### select

##### select使用

```c
select		int select(int nfds, fd_set readfds, fd_set writefds, fd_set errorfds, struct timeval timeout);
					input: nfds:select检查前nfds个文件描述符，起到一个缩小范围的作用；readfds:是否准备好读取的fd集合；writefds:是否准									 备好写入的fd集合；errorfds:类似前两者；timeout：超时时间
          output: select成功时返回就绪文件描述符的总数，e.g. fd=1,fd=2处都发生可读事件，则select返回2。
					select 检查readfds/writefds/errorfds在(0,nfds-1]处是否发生可读/可写/err 事件，如果没发生事件则阻塞，发生事件则返					回就绪文件描述符的总数。                                    
```

下面这个例子用来理解fd_set：

```c
取fd_set长度为1字节，fd_set中的每一bit可以对应一个文件描述符fd。则1字节长的fd_set最大可以对应8个fd。

（1）执行fd_set set;FD_ZERO(&set);则set用位表示是0000,0000。

（2）若fd＝5,执行FD_SET(fd,&set);后set变为0001,0000(第5位置为1)

（3）若再加入fd＝2，fd=1,则set变为0001,0011

（4）执行select(6,&set,0,0,0)阻塞等待---读取

（5）若fd=1,fd=2上都发生可读事件，则select返回，此时set变为0000,0011。注意：没有事件发生的fd=5被清空。
```

总结几点：

- select 可以监听的最大连接数为1024,即fd_set的最大长度为1024bit,在机器上定义为：`#define __DARWIN_FD_SETSIZE     1024`
- select的第一个参数用于界定select的监听范围，一般设定为maxfd+1，监听(0,maxfd]上的文件描述符
- select何时返回大于1？如果监听到多个文件描述符都发来消息，就会大于1。这是有可能出现的，在并发的场景下，如果有ABC3个人，第一次select监听到A的文件描述符，同时进行处理，BC同时再发来请求时就会阻塞，下一次select将会返回2.

##### 基于select的IO多路复用伪代码

```c
// 伪代码
/*
select一次监听可以监听到nready个描述符，这些描述符中有的是listenfd，说明是要建立连接的，有的是connfd，说明是要传输数据的。视情况分别进行处理，处理完nready个描述符后，select本次处理流程结束，进入下一次阻塞。
*/
int[] clientSet=new int[FD_SETSIZE];//存储 client connfd的数组,初始化均为-1
int maxLen=0;//clientSet中最高位的长度，目前是0，如果clientSet为[1,0,0,1,0,0,1,1,0,0,...]则maxLen=8
fd_set rset
fd_set allset  
  
listenfd = Socket();
Bind(listenfd,addr);
Listen(listenfd);

FD_SET(listenfd,&allset);//listenfd 加入allset

while(1){
  nready = select(maxLen,&rset,NULL,NULL,NULL);//监听rset
  if(rset.contains(listenfd)){//说明监听到此次为建立连接
  	connfd = Accept(listenfd);  
    clientSet.add(connfd)//将已经建立连接的connfd加入clientSet
    //maxLen更新
    nready--;  
  }
  for(int i=0;i<maxLen;i++){// 扫描(0,maxLen]的所有文件描述符
    int sockfd=clientSet[i];
    if (rset.contains(sockfd)){//说明监听到此次为request
      n=Read();
      if(n==0){//EOF
        Close(sockfd);
        clientSet[i]=-1;
      }else{
        //处理数据
        Write()
      }
    }
    if(--nready == 0){
      break;//此次select所监听到的所有文件描述符已经处理完
    }
  }

}
```

##### select的原理与不足

select实际将遍历fdList的过程转交给内核，这样避免多次read的系统调用：

![图片](https://tva1.sinaimg.cn/large/008i3skNgy1gwux4zc10gg30fa0b4x6p.gif)

select的过程中，用户看起来是阻塞的，此时内核在不断遍历fdList，fdList在select中也就是fd_set。

但是它存在以下问题：

1. fd_set的最大容纳fd数目有限制。
2. select运行时在内核中仍需不断遍历全部connfd以查看其是否就绪，效率低。
3. select无法直接返回已经就绪的connfd，用户仍需进行遍历才能找到哪些connfd可以进行read（上述伪代码的for循环），效率低。
4. 每次select时，都需要将fd_set从用户态复制到内核态，用户态和内核态频繁复制传递fd，数据开销大。 

#### poll

暂无

#### epoll

##### epoll的使用

```c
epoll_create 				int epoll_create(int size);
  									在内核中创建epoll实例并返回一个epoll文件描述符。 在最初的实现中，调用者通过 size 参数告知内核需要监听的文件										描述符数量。如果监听的文件描述符数量超过 size, 则内核会自动扩容。而现在 size 已经没有这种语义了，但是调用者										调用时 size 依然必须大于 0，以保证后向兼容性。
epoll_ctl 					int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
										向 epfd 对应的内核epoll 实例添加、修改或删除对 fd 上事件 event 的监听。op 可以为 EPOLL_CTL_ADD, 													EPOLL_CTL_MOD, EPOLL_CTL_DEL 分别对应的是添加新的事件，修改文件描述符上监听的事件类型，从实例上删除一个										事件。如果 event 的 events 属性设置了 EPOLLET flag，那么监听该事件的方式是边缘触发。
epoll_wait 					int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);  
										当 timeout 为 0 时，epoll_wait 永远会立即返回。而 timeout 为 -1 时，epoll_wait 会一直阻塞直到任一已注										册的事件变为就绪。当 timeout 为一正整数时，epoll 会阻塞直到计时 timeout 毫秒终了或已注册的事件变为就绪。										因为内核调度延迟，阻塞的时间可能会略微超过 timeout 毫秒。
```

##### 基于epoll的IO多路复用伪代码

利用epoll的程序和select的程序并没有多大区别，本质上都是IO多路复用。

```c
// 伪代码
/*
select一次监听可以监听到nready个描述符，这些描述符中有的是listenfd，说明是要建立连接的，有的是connfd，说明是要传输数据的。视情况分别进行处理，处理完nready个描述符后，select本次处理流程结束，进入下一次阻塞。
*/  
struct epoll_event ev;// epoll事件

listenfd = Socket();
setnonblocking(listenfd);//listenfd变为非阻塞
Bind(listenfd,addr);
Listen(listenfd);

epfd = epoll_create();//创建epoll实例
ev.events = EPOLLIN | EPOLLET;// 设置event为IN事件
ev.data.fd = listenfd;

epoll_ctl(epfd, EPOLL_CTL_ADD, listenfd, &ev)// 向 epfd 对应的内核epoll实例添加对 listenfd 上事件 ev 的监听
  
FD_SET(listenfd,&allset);//listenfd 加入allset


while(1){
  nfds = epoll_wait(epfd, events, 1, -1);// 监听epfd上的所有事件，nfds为事件数量，所有的就绪fd保存在events[n].data.fd中
  for (n = 0; n < nfds; ++n)
  {
    if (events[n].data.fd == listenfd)//处理连接请求
    {
      connfd = accept(listenfd, (struct sockaddr *)&cliaddr,&socklen);
      setnonblocking(connfd)
      ev.events = EPOLLIN | EPOLLET;
      ev.data.fd = connfd;
      epoll_ctl(kdpfd, EPOLL_CTL_ADD, connfd, &ev);//向 epfd 对应的内核epoll实例添加对 connfd 上事件 ev 的监听
      continue;
    }
    else //处理传输请求
    {
      handle(events[n].data.fd);// events[n].data.fd就是就绪的connfd
    }
  }
}
```



有几点不同：

1. epoll取消了fd_set的复制
2. epoll可以直接得到就绪的connfd，存放在`event.data.fd`中，不用特别去遍历

##### epoll的原理与改进

epoll针对select做了以下改进：

1. 内核中保存一份文件描述符集合，无需用户每次都重新传入，只需告诉内核修改的部分即可。(修改时通过epoll_ctl函数的EPOLL_CTL_ADD和EPOLL_CTL_DELETE完成)

2. 内核不再通过轮询的方式找到就绪的文件描述符，而是通过异步 IO 事件唤醒，epoll_wait阻塞时效率比select提高很多

   ![图片](https://tva1.sinaimg.cn/large/008i3skNgy1gwuyg8cq33g30fa0b4e81.gif)

3. 内核仅会将有 IO 事件的文件描述符(伪代码中的events[n].data.fd)返回给用户，用户无需遍历整个文件描述符集合。

## 提升程序容错能力

### 内核socket_buff空间不足

server端调用`read`，内核socket_buff空间不足时，会出现read一次读不完的现象：

```c
int read(int connfd, char* buf, int MAXLINE)
```

一次read读不完，server端就会发送多个TCP报文，这不是我们所期望的，因为client无法判别多个TCP报文是否属于一次request。

为此，引入`readline`函数，一次完整读取client传来的一行，以'\n'结尾(可能调用多次`read`)

```c
ssize_t ReadLine(int fd, void *buff, size_t maxlen)
```

### server断开再重连出现端口占用

正常连接状态下

​               有3条TCP连接

![image-20211114105123451](https://tva1.sinaimg.cn/large/008i3skNgy1gwehmxue9uj30n702a74n.jpg)

server断开

​			  仍保有一条TCP连接：

![image-20211114110113426](https://tva1.sinaimg.cn/large/008i3skNgy1gwehx721hwj30nc01gq2y.jpg)

​              server端的状态为:

![image-20211114110513492](https://tva1.sinaimg.cn/large/008i3skNgy1gwei1bt9jij30fl00ldfn.jpg)

server状态不为CLOSED，无法重新监听，通过`socketpot`设置`SO_REUSEADDR`即可解决

### server断开client一直request出现 broken pipe


client向server发送信息时要判断连接是否依然健在，否则会：write调用将数据放到TCP缓存区，TCP发送，server收到但是发现不存在连接，返回一个RST段，client收到RST段后将状态保存在TCP协议层，下一次再发送的时候，应用层会收到SIGPIPE信号，从而出现brokenPipe

# 基于udp协议的网络程序

# UNIX Domain 





























