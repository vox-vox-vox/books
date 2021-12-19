---
typora-copy-images-to: upload
---



编译环境相关

make dev  相当于在本地使用dev环境（dev的数据库，dev的mojito和dev的cadence），并不等于前段直接访问dev后端

make dev  指令可以将本地IDE的conf.yaml更新成conf-dev.yaml

```linux
cp config/conf.dev.yaml config/conf.yaml
```

前端

local----> 本地服务器------------因为本地环境用的是dev，所以和线上dev用的是同样的数据库与cadence等

dev ---->线上dev服务器--------



# ~/.zshrc    ~/.bash_profile

`~/.zshrc`是zsh的配置文件，`/.bash_profile`是bash的配置文件

`~/.local.rc` 是什么， source `~/.zshrc`是加载环境变量吗



# gin test 会自动运行所有前缀带test



# golang 相对路径和绝对路径

`./`是相对路径，代表当前目录



# gorm Raw 必须加Scan，否则不会执行，要想不加要用scan



# errors.New()可以用来自定义error

### 方式1：errors.New

    import "errors"
    func main() {
        var err error = errors.New("this is a new error")
        fmt.Println(err.Error())
            //this is a new error
        fmt.Println(err)
            //this is a new error
    }

### 方式2： fmt.Errorf
    err = fmt.Errorf("%s", "the error test for fmt.Errorf")
    fmt.Println(err.Error())



# shouldBind

1. 绑定参数结构体中名字首字母必须大写

2. get方法必须使用form格式，而不能使用json

# cURL 发送http请求

curl 'http://local.hdmap.momenta.works:8080/api/v1/workflow-defs?page=1&page_size=10' \
  -H 'Connection: keep-alive' \
  -H 'User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/92.0.4515.131 Safari/537.36' \
  -H 'Accept: */*' \
  -H 'Origin: http://local.hdmap.momenta.works:8001' \
  -H 'Referer: http://local.hdmap.momenta.works:8001/' \
  -H 'Accept-Language: zh-CN,zh;q=0.9' \
  -H 'Cookie: gr_user_id=abe97ef5-07d1-424a-8de7-d35ea8234746; id_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJjb3VudCI6MSwiZXhwIjoxNjI4OTE0Mzg2LCJ1c2VybmFtZSI6Imh1eGlhbyJ9.jaZgK9FG7utcCMZ7CJmNe2hkDRs21YdIa-HAWjvcRLE; id_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJjb3VudCI6MSwiZXhwIjoxNjI4OTE0Mzg2LCJ1c2VybmFtZSI6Imh1eGlhbyJ9.jaZgK9FG7utcCMZ7CJmNe2hkDRs21YdIa-HAWjvcRLE' \
  --compressed \
  --insecure





# swagger

![image-20210812163632250](https://tva1.sinaimg.cn/large/008i3skNly1gvkh96q91hj60mh0bumyd02.jpg)

![image-20210812162228149](https://tva1.sinaimg.cn/large/008i3skNly1gvkh93g0oij60cr06ewer02.jpg)



 生成swagger：swagger generate spec -o ./swagger.json

解析swagger.json，查看效果:swagger serve -F=swagger ./swagger.json  







一共增加了两个接口

1. PUT     /segment-tasks/:id/length     

   参数放在body中，可传入cell_length或region_length 

2. POST  /segment-tasks/region-lengths

   



# 匿名函数

```go
func(list []*shared.WorkflowExecutionInfo) ([]interface{}){
   data := make([]interface{}, 0)
   for _, v := range list{
      data = append(data, v)
   }
   return data
}(resp.Executions)
```

# cherry pick

用于解决只想把自己的提交上线，而不想管别人提交的情景

比如我提交了一个小bug，想快速上线，但是dev环境已经合了很多其他人的代码，我只想把我的快点上线到staging环境

可以先checkout到staging环境，然后git checkout -b feat/hx 新建一个和staging一模一样的分支，在这个分支上cherrypick

# nc-l用于查看请求是否已经正确发到了对应端口

例子：

```shell
nc -l 7900 # 监控7900端口
```



# byte[]



# mock测试

```go
package main

import (
	"testing"

	"github.com/ahuigo/requests"
	"gopkg.in/h2non/gock.v1"
)

func TestFoo(t *testing.T) {
	defer gock.Off()

	// mock response
	gock.New("http://m.com").
		Post("/bar").
		Reply(200).
		JSON(map[string]string{"foo": "bar"})

	// send request
	resp, err := requests.Post("http://m.com/bar")
	if err != nil {
		t.Fatal(err)
	}
    t.Log(resp.R.StatusCode)
	t.Log(resp.Text())
}
```



gock default 情况下无法拦截第二次http request，开启 `Persist()` 配置使其永久拦截，设置`Time()`配置使其规定拦截次数

https://github.com/h2non/gock/issues/48



## mock不匹配时同时还能访问真实地址的功能

我们开启mock后总会存在一些请求不想mock掉，而是真正的调用。设置`gock.EnableNetworking()`使得mock不匹配时可以绕过mock直接进行http请求，但是开启这个模式后会有bug，mock的地址对应的端口也必须有连接，换句话说我们不能mock自己随便编的一个网站，比如这样：

```go
➜  go_test go test mock2_test.go
package main

import (
        "testing"
        "github.com/ahuigo/requests"
        "gopkg.in/h2non/gock.v1"
)

func TestFoo(t *testing.T) {
        defer gock.Off()
        defer gock.DisableNetworking()

        // mock response
        gock.EnableNetworking()
        gock.New("http://123456").
                Get("/bar").
                Reply(200).
                JSON(map[string]string{"foo": "bar"})


        // send request
        resp, err := requests.Get("http://123456/bar")
        if err != nil {
                t.Fatal(err)
        }
        t.Log(resp.R.StatusCode)
        t.Log(resp.Text())
}
输出有问题：
--- FAIL: TestFoo (0.00s)
    mock2_test.go:25: GET http://123456/bar Get "http://123456/bar": dial tcp 0.1.226.64:80: connect: no route to host
FAIL
FAIL	command-line-arguments	0.011s
FAIL

改成能真正访问的就好了：
➜  go_test go test mock2_test.go
package main

import (
        "testing"

        "github.com/ahuigo/requests"
        "gopkg.in/h2non/gock.v1"
)

func TestFoo(t *testing.T) {
        defer gock.Off()
        defer gock.DisableNetworking()

        // mock response
        gock.EnableNetworking()
        gock.New("http://pm-ui.hdmap.momenta.works").
                Get("/bar").
                Reply(200).
                JSON(map[string]string{"foo": "bar"})


        // send request
        resp, err := requests.Get("http://pm-ui.hdmap.momenta.works/bar")
        if err != nil {
                t.Fatal(err)
        }
        t.Log(resp.R.StatusCode)
        t.Log(resp.Text())
}
```

感觉是一个bug，按理说我mock的地址不应该一定要求要是能访问到的才对。



# 请求超时

cadence内部bug，11.5版本之前不传runID会出现请求超时。传请求时传入上下文ctx，在里面规定TImeOut，就可以设定本次访问的超时时间，如果超过这个时间还没有收到resp就视为超时，ctx会在生成此条语句的时候同时设置一个deadline，deadline记录最晚的返回时间。

![Figure 1.1: How timeouts prevent long API calls](https://engineering.grab.com/img/context-deadlines-and-how-to-set-them/image5.jpg)

解决这个bug单纯设置超时时间久一点是没有用的，应当传入RunID



# kill 占用端口的进程

```shell
lsof -i :8080 # 找到占用8080的进程PID
kill PID
```



# 正则表达式入门

https://deerchao.cn/tutorials/regex/regex.htm





# ssh 

## 对称加密和非对称加密

SSH和telnet、ftp等协议主要的区别在于**安全性**

加密的方式主要有两种：

1. 对称加密（也称为秘钥加密）

所谓对称加密，指加密解密使用同一套秘钥。如下图所示：

![image-20210922134118801](/Users/huxiao/Library/Application Support/typora-user-images/image-20210922134118801.png)

2. 非对称加密（**公钥加密，私钥解密**）

对称加密的加密强度高，很难破解。但是在实际应用过程中不得不面临一个棘手的问题：**如何安全的保存密钥呢？**尤其是考虑到数量庞大的Client端，很难保证密钥不被泄露。**一旦一个Client端的密钥被窃据，那么整个系统的安全性也就不复存在**。为了解决这个问题，**非对称加密**应运而生。非对称加密有两个密钥：**“公钥”**和**“私钥”**。**公钥加密后的密文，只能通过对应的私钥进行解密。而通过公钥推理出私钥的可能性微乎其微。** **私钥是Server端独有**，这就保证了Client的登录信息即使在网络传输过程中被窃据，也没有私钥进行解密，保证了数据的安全性，这充分利用了非对称加密的特性。如下图所示：

<img src="https://tva1.sinaimg.cn/large/008i3skNly1gvkh2wlffbj60u00uf76u02.jpg" alt="image-20210922134614059" style="zoom: 50%;" />

client的公钥是server提供的。client根据提供的公钥，进行加密，然后server利用私钥进行解密

## 中间人攻击

上述流程会有一个问题：**Client端如何保证接受到的公钥就是目标Server端的？**，如果一个攻击者**中途拦截Client的登录请求**，向其发送自己的公钥，Client端用攻击者的公钥进行数据加密。攻击者接收到加密信息后再用自己的私钥进行解密，不就窃取了Client的登录信息了吗？这就是所谓的[中间人攻击](https://en.wikipedia.org/wiki/Man-in-the-middle_attack)。

<img src="https://tva1.sinaimg.cn/large/008i3skNly1gvkh360e5uj60ya0qc41202.jpg" alt="image-20210922135340717" style="zoom:50%;" />

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





# 快捷键

option+shift+↑↓=代码行移动

shift+方向键=代码选中

三指=代码选中

crtl+A=到行首

crtl+E=到行尾



# Json

空字符串不是一个合法的json，因为不知道会被反序列化成什么东西。



# shell

${变量}

$(指令)



```v
:se nonu
```



# 进程和线程

进程的地址空间

<img src="https://tva1.sinaimg.cn/large/008i3skNly1gvpapdnkmvj60zs0u0jta02.jpg" alt="image-20211023155223117" style="zoom:50%;" />

线程的地址空间





