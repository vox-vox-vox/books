---
typora-copy-images-to: upload
---

# File

## ext2文件系统

- 文件系统目录为树形
- 块大小：文件系统度量单位，4096表示4kb
- Inode表: 文件描述信息（ls-l显示的权限，文件大小等）存储在inode中。分配原则：大概每8k分配一个inode，1M的磁盘有128个inode。一个文件一个inode，一个大概128字节
- 数据块：某个分区块大小1024字节，某个文件2049字节，需要3个数据块存储，即使第三个块只存储了1字节也要占用一整块
- 块位图：每个bit代表块组中的一个块，1为可用，0为不可用
- 块组描述符：
- inode位图：每一位代表一个inode是否可用
- **符号链接文件**保存一个路径名，大小也就是路径名的长度
- mount：将某个存储介质挂载到某个目录
- stat(2)函数读取文件的inode，然后把inode中的各种文件属性填入一个struct stat结构体传出
- 硬链接
- 普通文件、目录文件、socket文件/文本文件、设备文件、网络文件、可执行文件

## 虚拟文件系统VFS

### 概念

linux内核再各种不同的文件系统格式之上定义了一个抽象层，使得文件、目录、读写访问等概念成为抽象层的概念，这个抽象层称为虚拟文件系统(VFS Virtual FileSystem)。如下图所示：

<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gw99uo47apj30u00u0juj.jpg" alt="image-20211109223411642" style="zoom:67%;" />

#### 文件描述符表

<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gwabvdjpw0j304i06st8k.jpg" alt="image-20211110202940426" style="zoom:50%;" />

**每个进程**在PCB中维护文件描述符表，文件描述符就是这个表的index。表中存放着指向file结构体的指针。

#### file结构体(不是C标准库的FILE结构体)

<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gwabtteqklj307407kq2y.jpg" alt="image-20211110202807655" style="zoom: 50%;" />

- **f_flags** File Status 读、写、追加、非阻塞等标志。可以在`open`时设定，也可以利用`fcntl`设定
- **f_ops** 当前读写位置
- **f_count** 引用计数： dup、fork等系统调用会导致多个文件描述符指向同一个file结构体，引用计数会增加。close的时候只减少引用计数，引用计数减到0时才真正关闭了文件。
- **f_op** file operation结构体(见下)
- **f_dentry** dentry结构体(见下)

#### file operation结构体

<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gwac1407foj305u08w74a.jpg" alt="image-20211110203511888" style="zoom:50%;" />

每个file结构体都指向 **同一个** file operation结构体。file operation 的成员都是函数指针，指向实现各种文件操作的内核函数。常规文件的file operation是同一个。

#### inode结构体

<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gwayn42mt4j307407kt8q.jpg" alt="image-20211111093728149" style="zoom:50%;" />

inode结构体保存着从磁盘分区的inode上读取到的inode信息，如所有者、文件大小、文件权限位等

#### inode_operation 结构体

<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gways5biztj306g0aiq32.jpg" alt="image-20211111094218809" style="zoom:50%;" />

存放函数指针，所指向的不是针对某一文件的操作函数，而是影响文件和目录布局的函数。属于同一文件系统的各个inode指向同一个inode_operation 结构体

#### dentry(directory entry)结构体

<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gway5hzqpgj303f05h3yf.jpg" alt="image-20211111092033081" style="zoom:70%;" />

**dentry是inode的索引**

directory entry：目录项

dentry结构体是一个树状cache，里面保存了inode和指向dentry的指针。为了减少读盘次数，内核缓存了目录结构，每次只要通过dentry读取inode即可。如果缓存失效，则仍要从磁盘读取inode

#### superblock结构体

<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gwazb0hju4j307205qjrd.jpg" alt="image-20211111094356123" style="zoom:50%;" />

保存着从磁盘超级块读上来的信息：文件系统类型，块大小等。s_root指向挂载(mount)区域



## 文件操作函数

### dup/dup2(重定向)

```c
dup			int dup(int oldfd);
				input: oldfd 原来的文件描述符
				output:	成功时返回新的文件描述符，失败时返回-1
				复制一个文件描述符，指向oldfd所指向的file结构体
dup2		int dup2(int oldfd, int newfd);
				input: oldfd 原先的文件描述符；newfd 新的文件描述符
				output: 成功时返回newfd，失败返回-1
				复制一个文件描述符，同时可以指定新的文件描述符的数值。如果新的文件描述符newfd此时已经打开，则先关闭再进行复制。看起来有
				点绕，实际上相当于把newfd指向的file结构体断开，设定其指向oldfd所指向的file结构体

```

demo与解析

```c
int main (void){
	int fd, save_fd;
	char msg[]="this is a test\n";
	fd =open("somefile",O_RDWR|O_CREAT,S_IRUSR|S_IWUSR);// 只读，不存在则创建
	if(fd<0){
		perror("open failed");
		exit(1);
	}
	save_fd = dup(STDOUT_FILENO);
	dup2(fd,STDOUT_FILENO);
	close(fd);
	write(STDOUT_FILENO,msg,strlen(msg));
	dup2(save_fd,STDOUT_FILENO);
	write(STDOUT_FILENO,msg,strlen(msg));
	close(save_fd);
	return 0;
}
```

<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gwazx2bx80j30k90hawfy.jpg" alt="image-20211111102138587" style="zoom:67%;" />



# IO

## 系统级IO

### 概念

#### 重定向

##### shell中的重定向

\>B  将标准输出(输出到屏幕)重定向至输出至B

\>> B 重定向&追加

\< B 以B作为标准输入，也许是以B中的内容作为标准输入

2>B 将 文件描述符表[2] 处的文件指针指向B

2>&1 将 文件描述符表[2] 处的文件指针指向 文件描述符表[1] 处指向的内容

##### 原理

重定向的本质是，进程的文件描述符指向新的file结构体。比如说之前文件描述符表[0] = stdin，经过输入重定向 (... \< B)，文件描述符表[0] = B

重定向在c语言中，利用dup2实现

### unbuffered IO函数

#### open/close

```c
open:				int open(const char *pathname, int flags, mode_t mode); 
						input: 	pathname:文件路径; flag:文件读写方式; mode:文件权限(可选)
						output:	文件描述符
close:			int close(int fd);
						input:	fd:要关闭的文件描述符
```

注意：

- open的参数flag就是file结构体中的f_flags

#### read/write

```c
read		ssize_t read(int fd, void *buf,size_t count);
				input: fd 文件描述符 ; buf 缓冲区 ; count 请求读取的字节数
				output: 成功时返回实际读取的字节数，出错时返回-1 并设置errno，达到文件末尾返回0
				从打开的设备或文件中读取数据，读上来的数据保存在buf中
				一些情况下，实际读取的字节数可能会小于请求读取的字节数，如：
				1. 读常规文件时提前到达末尾。比如count=100，实际只剩30个字节没有读，则返回30，下一次返回为0。
				2. q
				3. 网络IO读取
write		ssize_t write(int fd, const void *buf, size_t count);
				input: fd 文件描述符 ; buf 缓冲区 ; count 写入的字节数
				output: 成功时返回实际写入的字节数，出错时返回-1 并设置errno，达到文件末尾返回0
				从buf向打开的文件或设备fd中写数据
```

注意：

- read/write 操作在常规文件不会发生阻塞，而操作在设备文件(终端、显示器)或网络文件时可能会发生阻塞。
- 通过设定文件f_flag，可以设置非阻塞读写。非阻塞读写可以采用 **轮询** 同时监控多个设备：

​		<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gwb6hrh8awj30aa06pq36.jpg" alt="image-20211111140909735" style="zoom: 67%;" />

​		防止CPU空转，可以采用 **sleep+轮询** 方式：

​		<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gwb6k0v8j1j30a20760t0.jpg" alt="image-20211111141120746" style="zoom:67%;" />

​		最后，可以用**select**解决		

#### 调用read时发生了什么？

用户read一个文件描述符，read通过系统调用进入内核，找到这个文件描述符指向的file结构体，找到file结构体指向的file_operations结构体，调用它的read成员所指向的内核函数完成用户请求。

用户read

read对应的内核函数

#### fcntl

```c
fcntl		int fcntl(int fd,int cmd)
				input: fd 文件描述符；cmd 文件权限操作，如F_GETFL(译为getFlag，即得到文件的Status Flag)
				output:
				不需要重新open文件，可以直接设置读、写、追加、非阻塞等标志 (File Status Flag)
```

#### lseek

```c
lseek		off_t lseek(int fd, off_t offset, int whence)
				input:
				output:
				移动读写位置,同fseek相同
```

#### mmap

#### ioctl

## C语言IO

### 概念

#### FILE 

是C标准库中定义的 **结构体** 类型，包含**文件描述符**、IO缓冲区、当前读写位置等信息。不必知道FILE具体有哪些成员，FILE* 作为不透明指针在C标准IO库函数中传递

#### stdin/stdout/stderr

特殊的FILE*。在程序启动时会将终端设备打开三次，并分别赋给stdin、stdout、stderr这三个 FILE\*指针。

stdin：标准输入，对应键盘

stdout：标准输出，对应显示器

stderr：标准错误，对应显示器。利用重定向可以将标准输出定向到常规文件中，从而和标准错误区分

#### C标准IO缓冲区

C标准库为 **每一个打开的文件** 分配一个C标准IO缓冲区加速读写操作，**一个文件对应一个标准IO缓冲区**，通过文件的FILE结构体可以找到这个缓冲区。

根据文件的不同，C标准IO缓冲区也分为三种类型：

- 全缓冲：缓冲区写满了就写回内核(flush)。常规文件通常是全缓冲的
- 行缓冲：用户程序中有换行符就把这一行写回内核/缓冲区满了就写回内核(flush)。对应终端设备时通常是行缓冲。行缓冲有3种情况会flush，分别是: 1.有换行符'\n' 2.IO缓冲区写满 3.用户程序调用库函数从无缓冲文件读取/从行缓冲文件读取且这次读操作会引发系统调用从内核中取数据
- 无缓冲：每次调用库函数都要经过系统调用写回内核(flush)。标准错误是无缓冲的，产生的错误信息可以尽快输出到设备

下图以`fputs`和`fgets`示意了C标准I/O缓冲区的作用。注意到，`fputs`和`fgets`本身也需要提供用户缓冲区`buf1` 和`buf2`，要和C标准IO缓冲区(图中`I/O buffer`)予以区分。

<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gw8zv5gcwuj30s40bq3z2.jpg" alt="image-20211109164840462" style="zoom:50%;" />

### 标准IO库函数

#### fopen/fclose

```c
fopen       FILE *fopen(const char *path, const char *mode);
            input: path 路径名 ; mode 操作方式，可传入"r"(只读) "w"(只写) 等
            output: 成功返回文件指针，出错返回NULL并设置errno
            打开文件
fclose      int flose(FILE *fp);
            input: fp 文件指针
            output:
            关闭文件
```

#### fgetc/fputc

```c
fgetc       int fgetc(FILE *stream);
            input: FILE 文件的指针，也可称为流指针
            output: 成功时返回读到的字节，出错/读到文件末尾时返回EOF。本来应该返回unsigned 
            				char，但是由于函数原型中返回是int，所以要将char转为int再返回。这是为了										兼容EOF
            从指定的文件中读一个字节。
getchar     int getchar(void);
            input: -
            output: 成功时返回读到的字节，出错/读到文件末尾时返回EOF
            从标准输入中读一个字节，相当于fgetc(stdin)
fputc       int fputc(int c,FILE *stream);
            input: FILE 文件的指针，也可称为流指针 ; c 输入的字节
            output: 成功时返回写入的字节，出错/读到文件末尾时返回EOF
            向指定的文件中写入一个字节
putchar     int putchar(int c);
            input: 待写入的字符串
            output: 成功时返回写入的字节，出错/读到文件末尾时返回EOF
            向标准输出中写入一个字节，相当于fputc(c,stdout)
```

**注意**：

从终端设备读时，用户输入一般字符不会使`getchar/fgetc(stdin)`返回，而是发生阻塞。只**有换行符'\n'** 或 **文件末尾EOF** 才会返回

#### fseek/ftell/rewind

```c
fseek       int fseek(FILE *stream, long offset, int whence);
            input:FILE 文件的指针，也可称为流指针 ; offset 移动量，负值代表向前、正值代表
             			向后 ;whence SEEK_END ; SEEK_CUR ; SEEK_END 
            output:成功时返回0，错误返回-1并设置errno
            任意移动读写位置
ftell     	int ftell(FILE *stream);
         		input: FILE 文件的指针
         		output: 返回当前读写位置，错误返回-1并设置errno
rewind    	rewind(FILE *stream);
         		重新将读写位置移到开头
```

#### fgets/fputs

```c
fgets       char *fgets(char *s,int size,FILE *stream);
            input: s 缓冲区首地址 ; size 缓冲区长度 ; FILE 文件的指针，也可称为流指针
            output: 成功时s指向哪返回的指针就指向哪，出错/读到文件末尾时返回NULL
            从FILE中读取以'\n'结尾的一行(包括'\n')存到缓冲区s中，并在后面加上一个'\0'组成完整的字符
            串
gets        int gets(char *s);
            NEVER USE!!!!!
            用户提供一个缓冲区，却不能指定缓冲区大小，可能通过标准输入提供任意长字符串，导致缓冲区溢出  
fputs       int fputc(const char *s,FILE *stream);
            input: FILE 文件的指针，也可称为流指针 ; s 输入的字符串
            output: 成功时返回非负整数，出错/读到文件末尾时返回EOF
            向指定的文件中写入一个字符串
puts        int puts(const char *s);
            input: 字符串
            output: 成功时返回非负整数，出错/读到文件末尾时返回EOF
            向标准输入中写入一个字符串
```

**注意**：

1. 对终端来说，不输入换行符时`fgets`会阻塞
2. `fgets`本质上是将数据由C标准IO缓冲区读到用户自定义缓冲区s；`fputs`本质上是将数据由用户缓冲区s读到C标准IO缓冲区

#### printf/scanf

格式化IO 格式化字符串<=====>参数

```c
printf    int printf(const char *format, ...);
         	将输入的参数通过format格式化，打印到标准输出（屏幕）
scanf     int scanf(const char *format, ...);
         	从标准输入读字符，按格式化format转换输入的字符，并赋给后面的参数，后面的参数必须传地址
```

#### perror/stderror/errno

```c
perror      void perror(const char *s);
            input: 用户自定义的字符串
            output: -
            将错误信息打印到标准错误输出，首先打印s所指的字符串，然后打印 ":" ，然后根据errno
            的值打印错误原因
strerror    char *strerror(int errnum);
            input: 错误码
            output: 错误码对应的字符串
            将错误码打印成对应字符串
errno       int
            错误码，全局变量，所有错误码都是正整数
```

