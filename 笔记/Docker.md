# Docker architecture

Docker 使用客户端-服务器架构。 Docker 客户端与 Docker 守护进程对话，后者负责构建、运行和分发 Docker 容器的繁重工作。 Docker 客户端和守护程序可以运行在同一系统上，或者您可以将 Docker 客户端连接到远程 Docker 守护程序。 Docker 客户端和守护进程使用 REST API、UNIX 套接字或网络接口进行通信。 另一个 Docker 客户端是 Docker Compose，它允许您使用由一组容器组成的应用程序。

![Docker Architecture Diagram](https://docs.docker.com/engine/images/architecture.svg)

## Docker daemon

Docker 守护进程 (`dockerd`) 侦听 Docker API 请求并管理 Docker 对象，例如镜像、容器、网络和卷。 守护进程还可以与其他守护进程通信以管理 Docker 服务。

## Docker client

Docker 客户端 (`docker`) 是许多 Docker 用户与 Docker 交互的主要方式。 当您使用诸如 docker run 之类的命令时，客户端会将这些命令发送到 dockerd，后者会执行这些命令。 docker 命令使用 Docker API。 Docker 客户端可以与多个守护进程通信。

## Docker Desktop

<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gxgqthhbw9j309606yq32.jpg" alt="image-20211217130030771" style="zoom:50%;" />

Docker Desktop 是一款易于安装的应用程序，适用于您的 Mac 或 Windows 环境，使您能够构建和共享容器化应用程序和微服务。 Docker Desktop 包括 Docker 守护进程 (dockerd)、Docker 客户端 (docker)、Docker Compose、Docker Content Trust、Kubernetes 和 Credential Helper。

## Docker registries

Docker 仓库存储 Docker 镜像。 Docker Hub 是一个任何人都可以使用的公共仓库，Docker 默认配置为在 Docker Hub 上查找镜像。 您甚至可以运行自己的私有仓库。

当您使用 docker pull 或 docker run 命令时，所需的镜像将从您配置的仓库中提取。您使用 docker push 命令时，您的镜像将被推送到您配置的仓库。

## Docker 对象

### image（镜像）

镜像是一个只读模板，包含创建 Docker 容器的说明。 通常，一个镜像基于另一个镜像，并带有一些额外的自定义。 例如，您可以构建一个基于 ubuntu 镜像的镜像，在其中安装 Apache Web 服务器和您的应用程序，以及使您的应用程序运行所需的配置详细信息。

您可以创建自己的镜像，也可以仅使用其他人创建并在仓库中发布的镜像。 要构建您自己的镜像，您需要使用简单的语法创建一个 Dockerfile，用于定义创建镜像和运行镜像所需的步骤。 Dockerfile 中的每条指令都会在镜像中创建一个层。 当您更改 Dockerfile 并重建镜像时，只会重建那些已更改的层。 与其他虚拟化技术相比，这是使镜像如此轻巧、小巧和快速的部分原因。

### container（容器）

容器是镜像的可运行实例。 您可以使用 Docker API 或 CLI 创建、启动、停止、移动或删除容器。 您可以将容器连接到一个或多个网络，为其附加存储，甚至可以根据其当前状态创建新镜像。

默认情况下，容器与其他容器及其主机相对隔离。 您可以控制容器的网络、存储或其他底层子系统与其他容器或主机之间的隔离程度。

容器由其镜像以及您在创建或启动它时提供给它的任何配置选项定义。 当容器被移除时，未存储在持久存储中的对其状态的任何更改都会消失。

简而言之，容器是：

- 一个镜像的可运行实例。 您可以使用 DockerAPI 或 CLI 创建、启动、停止、移动或删除容器。
- 可以在本地机器、虚拟机上运行，也可以部署到云端。
- 可移植（可以在任何操作系统上运行）
- 容器彼此隔离并运行自己的软件、二进制文件和配置。

# Docker 网络模式

### 4.1 Host 模式

等价于Vmware中的桥接模式，当启动容器的时候用host模式，容器将不会虚拟出自己的网卡，配置自己的IP等，而是使用宿主机的IP和端口。但是容器的其他方面，如文件系统、进程列表等还是和宿主机隔离的。

### 4.2 Container 模式

Container 模式指定新创建的容器和已经存在的一个容器共享一个 Network Namespace，而不是和宿主机共享。新创建的容器不会创建自己的网卡，配置自己的IP，而是和一个指定的容器共享IP、端口范围等。同样，两个容器除了网络方面，其他的如文件系统、进程列表等还是隔离的。两个容器的进程可以通过lo网卡设备通信。

### 4.3 None 模式

None 模式将容器放置在它自己的网络栈中，并不进行任何配置。实际上，该模式关闭了容器的网络功能，该模式下容器并不需要网络（例如只需要写磁盘卷的批处理任务）。



# Docker volume

每个容器每次启动时都是从镜像定义开始的。 虽然容器可以创建、更新和删除文件，但是当容器被移除并且所有更改都与该容器隔离时，这些更改将丢失。 有了卷（volume），我们可以改变这一切。

卷提供了将容器的特定文件系统路径连接回主机的能力。 如果挂载了容器中的目录，则主机上也会看到该目录中的更改。 如果我们在容器重新启动时挂载相同的目录，我们会看到相同的文件。

## named volume

开辟一个带有名字的“命名卷”（named volume），视为简单的数据桶。 Docker 维护磁盘上的物理位置，您只需要记住卷的名称。 每次使用该卷时，Docker 都会确保提供正确的数据。

`docker volume create todo-db` 创建一个name为todo-db的卷

`docker run -dp 3000:3000 -v todo-db:/etc/todos getting-started` 将使用`todo-db`这个卷，并将其映射至容器的`/etc/todos`中

`docker volume inspect todo-db ` 可以查看`named volume`在宿主机是怎么存放的

![image-20211218201445026](https://tva1.sinaimg.cn/large/008i3skNgy1gxi8zmqs5pj30d405ot8t.jpg)

## bind mounts

和`named volume`相比，`bind mounts` 适用于想知道具体挂载地点的场景

![image-20211218205148613](https://tva1.sinaimg.cn/large/008i3skNgy1gxia25hgiaj30qs05xaab.jpg)

## 挂载源代码

将源代码挂载至docker，就可以进入快速开发模式，对源代码的更改在容器内会马上生效



# Dockerfile

Docker build 根据Dockerfile 进行构建

# Docker compose

每个容器应当只做一个事情，这样我们可以做到隔离地更改

# Docker 指令

## 记录一些不知道该放在哪里的信息

比如-name指定容器名字，后面再 start 这个容器就不用查 id 了；-v挂载文件，将你本地的代码挂载进 docker；或者-p映射端口，将 docker 的端口映射到本机，以便提供http 等服务。

## pull

拉取镜像

![image-20211130220326052](https://tva1.sinaimg.cn/large/008i3skNgy1gwxiz4vyjrj30x005oq3t.jpg)

## tag

`docker tag` 标记某一镜像，使其加入某一仓库

`docker tag getting-started voxvoxvox/getting-started`

将`getting-started`标记为`voxvoxvox/getting-started` 镜像。

<img src="/Users/huxiao/Library/Application%20Support/typora-user-images/image-20211218175148693.png" alt="image-20211218175148693" style="zoom:80%;" />

`voxvoxvox/getting-started`一定要和仓库新建的repository同名，否则无法进行`push`

## push

`docker push` 将本地仓库push至线上仓库

## images

列出本地镜像

![image-20211130220613526](https://tva1.sinaimg.cn/large/008i3skNgy1gwxj1zxy45j317q06m3zz.jpg)

## ps

列出容器

`-a`表示列出所有，包括没运行的：

![image-20211130230516757](https://tva1.sinaimg.cn/large/008i3skNgy1gwxkrfq7jrj319c098goc.jpg)

## exec

`docker exec` 在运行的容器中执行命令

`docker exec <container-id> cat /data.txt`

在运行的容器中（container-id对应的容器）执行命令 `cat /data.txt`

## start

`docker start -i CONTAINER_ID` 启动一个指定ID的container

## stop

`docker stop <the-container-id>` 关闭一个指定ID的container

## run

`docker run` 快速启动

既然是 ubuntu，我们可能想进入系统，安装软件，做一些操作。这就需要-it 参数，执行 docker run -it ubuntu:16.04 /bin/bash，可以发现我们得到了一个 shell，你可以正常的安装软件或者使用了,你所做的一切都在容器内部，不影响你的系统。

`run = create + start`

`docker run -it -d --name halo -p 8090:8090 -v ~/.halo:/root/.halo --restart=always halohub/halo` 这条指令代表

`-it` 以交互模式运行，并返回一个伪终端

`-d` 后台运行

`--name` 为容器指定名称 

`-p` 指定端口映射，规则为：`宿主端口:容器端口`

`-v`  将宿主机目录挂载到容器目录，规则为：` /宿主机目录:/容器目录`

## build

`docker build` 创建镜像

`docker build -t getting-started .` 

`-t` 后加 tag名

`.` 表示将在当前目录寻找Dockerfile

## rm 

`docker rm` 删除一个或多个容器

`docker rm -f <container-id>` 

# Docker 原理

主要解决运行环境问题

虚拟机是资源层面的隔离，内存，CPU分配；有独立的操作系统

容器化是应用层面的隔离，硬件资源共享，10M--300M，很小，共享宿主机内核

容器化解决的问题：1. 环境问题 2. 轻量化

namespace

# Linux 容器化技术

容器=进程



由于虚拟机存在这些缺点，Linux 发展出了另一种虚拟化技术：Linux 容器（Linux Containers，缩写为 LXC）。

**Linux 容器不是模拟一个完整的操作系统，而是对进程进行隔离。**或者说，在正常进程的外面套了一个[保护层](https://opensource.com/article/18/1/history-low-level-container-runtimes)。对于容器里面的进程来说，它接触到的各种资源都是虚拟的，从而实现与底层系统的隔离。

由于容器是进程级别的，相比虚拟机有很多优势。

**（1）启动快**

容器里面的应用，直接就是底层系统的一个进程，而不是虚拟机内部的进程。所以，启动容器相当于启动本机的一个进程，而不是启动一个操作系统，速度就快很多。

**（2）资源占用少**

容器只占用需要的资源，不占用那些没有用到的资源；虚拟机由于是完整的操作系统，不可避免要占用所有资源。另外，多个容器可以共享资源，虚拟机都是独享资源。

**（3）体积小**

容器只要包含用到的组件即可，而虚拟机是整个操作系统的打包，所以容器文件比虚拟机文件要小很多。

总之，容器有点像轻量级的虚拟机，能够提供虚拟化的环境，但是成本开销小得多。

# Docker questions

1. ![image-20211217142551567](https://tva1.sinaimg.cn/large/008i3skNgy1gxgta94f2tj3041025t8i.jpg)这种名字有什么用？

2. `docker run -it ubuntu ls /` 

   和

   `docker run ubuntu ls /` 

   有什么区别？`-it` 后接执行语句是不是就无法进入交互模式？
   
3. 既然代码可以挂载至本地，那么docker build 所构建的镜像是否与本地内容有关？本地代码的改变是否会导致镜像的改变？还是说镜像中保存的只是一份快照？

​		镜像更像一个开发环境，而不涉及到开发环境中的数据。

​		容器是基于镜像（开发环境）的实例化，挂载卷(程序代码+数据)+镜像(开发环境)=容器

