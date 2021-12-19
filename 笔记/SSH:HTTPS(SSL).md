> 所有的黑客拦截都是为了得到明文密码，否则无法向server发送。无法直接向server发送加密后的密码，只能通过定制的client在输入框里输入明文
# 对称、非对称加密

## 对称加密和非对称加密

SSH和telnet、ftp等协议主要的区别在于**安全性**

加密的方式主要有两种：

### 对称加密（也称为秘钥加密）

所谓对称加密，指加密解密使用**同一套秘钥**。如下图所示：

<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gwslw748mfj311a0ou414.jpg" alt="image-20210922134118801" style="zoom: 50%;" />

对称加密的算法是公开的，唯一的安全性仅存在于密钥的保密

### 非对称加密（**公钥加密，私钥解密**）

为了解决这个问题，**非对称加密**应运而生。非对称加密有两个密钥：**“公钥”**和**“私钥”**。**公钥加密后的密文，只能通过对应的私钥进行解密。而通过公钥推理出私钥的可能性微乎其微。** **私钥是Server端独有**，这就保证了Client的登录信息即使在网络传输过程中被窃据，也没有私钥进行解密，保证了数据的安全性，这充分利用了非对称加密的特性。如下图所示：

<img src="https://tva1.sinaimg.cn/large/008i3skNly1gvkh2wlffbj60u00uf76u02.jpg" alt="image-20210922134614059" style="zoom: 50%;" />

client的公钥是server提供的。client根据提供的公钥，进行加密，然后server利用私钥进行解密

**RSA算法是一种经典的非对称加密**

### 对称加密vs非对称加密

- 二者不同：对称加密可用于双向加密解密，而非对称加密只能用于单向发送，因为私钥只能解密，公钥只能加密

- 对称加密优点：

  对称加密的加密解密**速度快**，效率高。

- 对称加密缺点：
  - **密钥管理难度大**。对称加密双方密钥相同（如都是”6368616e676520746869732070617373”），如何**提前约定好**成为很大的问题。采用秘钥分发的方式，会对密钥管理造成很大负担，如果n个人彼此通信，需要(n-1)n/2个密钥，这称为n^2问题，而非对称加密只需要每个人保存好自己的一对公钥私钥，也就是需要n对密钥。
  - **密钥不安全**。一旦对称加密的密钥被捕获，系统安全性不复存在，而非对称加密则只有得到私钥才能对数据进行解密。

**实际中，一般采用非对称加密进行认证，采用对称加密进行数据加密传输**

## 中间人攻击

上述流程会有一个问题：**Client端如何保证接受到的公钥就是目标Server端的？**，如果一个攻击者**中途拦截Client的登录请求**，向其发送自己的公钥，Client端用攻击者的公钥进行数据加密。攻击者接收到加密信息后再用自己的私钥进行解密，不就窃取了Client的登录信息了吗？这就是所谓的[中间人攻击](https://en.wikipedia.org/wiki/Man-in-the-middle_attack)。

<img src="https://tva1.sinaimg.cn/large/008i3skNly1gvkh360e5uj60ya0qc41202.jpg" alt="image-20210922135340717" style="zoom:50%;" />

# SSH

## 基于密码的ssh连接

采用上述非对称加密的方式，即：

<img src="https://tva1.sinaimg.cn/large/008i3skNly1gvkh39pu1jj60u00uf76u02.jpg" alt="image-20210922134614059" style="zoom:33%;" />

其中在第一步时可能会出现中间人攻击，所以要提醒client是否信任对方server，会弹出来一堆fingerprint之类的认证：

```shell
　　$ ssh user@host
　　The authenticity of host 'host (192.168.100.1)' can't be established. #auth不能被确保
　　RSA key fingerprint is 2d:37:16:58:4d:28:c2:42:2d:37:16:58:4d.
　　Are you sure you want to continue connecting (yes/no)?
```

把责任推给用户，确认之后就可以登录了。该server也就被标记为可信任的，存放在`.ssh/know_hosts`中。所以所有连接过的里面都会有保存：

![image-20210922141725140](https://tva1.sinaimg.cn/large/008i3skNly1gvkh3g3r3pj60ix0a5q5o02.jpg)

如果server端出现变化，比如我的服务器从centOS换成ubuntu，重启了server的实例，再ssh连接的时候就会出现这样的提醒：

![image-20211211211620462](https://tva1.sinaimg.cn/large/008i3skNgy1gxa7fk59kcj30gp06p3zk.jpg)

根据它的提示，fingerprint发生改变，server已经不是以前连接的server了，可能出现了中间人攻击！当然我们知道其实不是中间人攻击，这个问题的解决办法就是直接把以前的fingerprint删了就行了。

具体连接过程与细节：

1. 直接用ssh指令即可连接

```shell
ssh cyk@172.21.65.94
```

2. 利用scp从服务器上下载

```shell
scp username@host:dir1 dir2 将dir1上的内容传输到dir2上
scp cyk@172.21.65.94:/home/cyk/pprof/pprof.main.samples.cpu.003.pb.gz ~/pprof
```

3. 利用scp上传到服务器

```go
scp /path/filename username@servername:/path 
```



## ssh 基于密钥的连接

### 为什么基于密码的ssh不安全

**因为要输入密码，将密码传送给server，只要存在这个过程就怎么都是不安全的**。有可能ssh的时候，确实连到了中间人，然后把密码发给它了，一切gg，所以要想追求绝对安全，**只能寻求一种不输入固定密码给server的方法**

### 基于密钥的连接

1. 利用`ssh-keygen -t rsa`客户端生成一对公钥和私钥，公钥保存在`id_rsa.pub`，私钥保存在`id_rsa`
2. 将公钥中的内容，存到server的`authorized_keys`
3. client再次发送ssh连接请求，server收到后会去`authorized_keys`中查找，有匹配的就随机生成一个字符串“如qwerty”，根据匹配的公钥加密发送到client
4. Client端通过私钥进行解密得到随机数R，然后对随机数R和本次会话的SessionKey利用MD5生成摘要Digest1，发送给Server
5. Server端会也会对R和SessionKey利用同样摘要算法生成Digest2。
6. Server端会最后比较Digest1和Digest2是否相同，完成认证过程

## 实战--配置ssh密钥连接github

1. 生成ed25519加密算法 的 密钥对，保存在.ssh下，键入passphrase( 相当于又加了一层保障 )

> With SSH keys, if someone gains access to your computer, they also gain access to every system that uses that key. To add an extra layer of security, you can add a passphrase to your SSH key. You can use `ssh-agent` to securely save your passphrase so you don't have to reenter it

一路enter，全接受默认配置(默认就是在`.ssh`下)，passphrase可以不用

![image-20210922175100411](https://tva1.sinaimg.cn/large/008i3skNly1gvkh3lqxpkj60de029t8u02.jpg)

生成密钥对如下

![image-20210922174933712](https://tva1.sinaimg.cn/large/008i3skNly1gvkh3oo08rj60df03c3yp02.jpg)

2. (可跳过)将公钥添加到ssh-agent上，ssh-agent的作用是如果passphrase不为空，可以再设置一个，避免重复输入密码：

> 因为输入密码可能很乏味，许多用户更愿意在每个本地登录会话中输入一次。存储未加密密钥最安全的地方是程序内存，在类 Unix 操作系统中，内存通常与[进程](https://en.wikipedia.org/wiki/Process_(computing))相关联。普通的 SSH 客户端进程不能用于存储未加密的密钥，因为 SSH 客户端进程只持续远程登录会话的持续时间。因此，用户运行一个名为**ssh-agent**的程序，该程序在本地登录会话期间运行，将未加密的密钥存储在内存中，并使用[Unix 域套接字](https://en.wikipedia.org/wiki/Unix_domain_socket)与 SSH 客户端通信。

3. 测试ssh连接

```shell
$ ssh -T git@github.com
```

此时通了，就说明连接好了：

![image-20210922180656884](https://tva1.sinaimg.cn/large/008i3skNly1gvkh3tsx3wj60is00x0so02.jpg)

4. 将git的提交等操作换为ssh方式

修改`.git/config`文件，将其方式改为ssh

![image-20210922180811396](https://tva1.sinaimg.cn/large/008i3skNgy1gvkh12657yj60cp06jwep02.jpg)

改为4

![image-20210922181018594](https://tva1.sinaimg.cn/large/008i3skNgy1gvkh1g6992j60cj06kjri02.jpg)

之后就成功啦！

![image-20210922181103901](https://tva1.sinaimg.cn/large/008i3skNly1gvkh4mfsgnj60p308fwfi02.jpg)



# HTTPS/SSL

HTTP+SSL=HTTPS

HTTPS在传统的HTTP和TCP之间加了一层用于`加密解密的SSL/TLS层`（安全套接层Secure Sockets Layer/安全传输层Transport Layer Security）层。使用HTTPS必须要有一套自己的数字证书（**包含公钥和私钥**）。

采用**非对称加密握手，对称加密通信**的方式，非对称加密握手的目的是为了生成对称加密需要的密钥。

握手类似3次握手：

- client给出协议版本号、一个生成的随机数A（Client random），以及client支持的加密方法。
- server确认双方使用的加密方法，并给出**数字证书**+公钥、以及一个服务器生成的随机数B（Server random）。
- Client确认数字证书有效（相当于认证fingerprint），然后生成一个新的随机数C（Premaster secret），并使用数字证书中的公钥，加密这个随机数C，发给server。
- server使用自己的私钥解密client发来的随机数C（即Premaster secret）。
- client和server利用随机数A，B，C对接下来的所有对话进行对称加密。

<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gwsom0z8voj31bs0u0jtw.jpg" alt="image-20211126173038462" style="zoom:50%;" />



需要注意：

- 生成对话密钥一共需要三个随机数。

- 握手之后的对话使用"对话密钥"(A+B+C)加密（对称加密），服务器的公钥和私钥只用于加密和解密"对话密钥"（非对称加密），无其他作用。

- 服务器公钥放在服务器的数字证书之中。















