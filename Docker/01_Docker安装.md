# Docker 安装

## Docker能做什么

Docker是一个用于开发，发布和运行应用程序的开放平台。通俗的讲，Docker类似于集装箱。在一艘大船上，各种货物要想被整齐摆放并且相互不受到影响，我们就需要把各种货物进行集装箱标准化。有了集装箱，我们就不需要专门运输水果或者化学用品的船了。我们可以把各种货品通过集装箱打包，然后统一放到一艘船上运输。Docker要做的就是把各种软件打包成一个集装箱（镜像），然后分发，且在运行的时候可以相互隔离。

## 安装Docker

Docker是跨平台的解决方案，它支持在当前各个平台安装。

### 卸载已有的Docker

https://docs.docker.com/engine/install/ubuntu/#uninstall-docker-engine

如果你已经安装过旧版的Docker，可以先执行以下命令卸载Docker

```bash
# 停止docker服务
$ sudo systemctl stop docker
# 删除docker所有包
$ sudo apt autoremove docker docker-ce docker-engine  docker.io  containerd runc
# 查看docker是否卸载干净
$ dpkg -l | grep docker
$ dpkg -l |grep ^rc|awk '{print $2}' |sudo xargs dpkg -P
# 删除相关插件
$ apt autoremove docker-ce-*
# 删除docker配置目录
$ rm -rf /etc/systemd/system/docker.service.d
$ rm -rf /var/lib/docker
# 确定以下
docker --version
```

### 安装Docker

https://docs.docker.com/engine/install/ubuntu/#install-using-the-convenience-script

```bash
$ curl -fsSL https://get.docker.com -o get-docker.sh
$ sudo sh get-docker.sh
Executing docker install script, commit: 7cae5f8b0decc17d6571f9f52eb840fbc13b2737
<...>
$ sudo systemctl start docker 
```

> 安装完成后默认 docker 命令只能以 root 用户执行，如果想允许普通用户执行 docker 命令，需要执行以下命令 `sudo groupadd docker && sudo gpasswd -a ${USER} docker && sudo systemctl restart docker`，执行完命令后，退出当前命令行窗口并打开新的窗口即可。

这里有一个国际惯例，安装完成后，我们需要使用以下命令启动一个hello world的容器。

```bash
$ sudo docker run hello-world

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

### 为Docker设置代理

**设置系统代理**

```bash
$ vim ~/.bashrc
export HTTP_PROXY=127.0.0.1:8080
export HTTPS_PROXY=127.0.0.1:8080
export NO_PROXY=localhost,127.0.0.1
$ source ~/.bashrc
```

**设置Docker代理**

```bash
$ sudo mkdir -p /etc/systemd/system/docker.service.d
$ sudo nano /etc/systemd/system/docker.service.d/http-proxy.conf
[Service]
Environment="HTTP_PROXY=127.0.0.1:8080"
Environment="HTTPS_PROXY=127.0.0.1:8080"
Environment="NO_PROXY=localhost,127.0.0.1"
$ sudo systemctl daemon-reload
$ sudo systemctl restart docker
```

**设置Docker Compose代理**

```bash
# 方法1
# 如果使用 Docker Compose，还需要确保 Docker Compose 使用代理。可以在 Docker Compose 文件中添加环境变量，或者在运行 Docker Compose 命令时指定代理：
version: '3'
services:
  web:
    image: nginx
    environment:
      - HTTP_PROXY=http://proxy.example.com:8080
      - HTTPS_PROXY=https://proxy.example.com:8080
      - NO_PROXY=localhost,127.0.0.1

# 方法2
$ HTTP_PROXY=http://proxy.example.com:8080 HTTPS_PROXY=https://proxy.example.com:8080 NO_PROXY=localhost,127.0.0.1 docker-compose up
```

**Docker创建docker用户组，应用用户加入docker组**

```bash
1、添加docker group：
sudo groupadd docker

2、将当前用户添加到docker组：
sudo gpasswd -a ${USER} docker

3、重启docker服务：
sudo service docker restart
```

```bash
$ sudo groupadd docker
$ sudo usermod -aG docker ${USER}
$ sudo systemctl restart docker
$ su root # 切换到root用户
$ su ${USER} # 在切换到原来的应用用户以上才生效
```

## 容器技术原理

提起容器就不得不说chroot，因为chroot是最早的容器雏形。chroot意味着切换根目录，有了chroot就意味着我们可以把任何目录更改为当前进程的根目录，这与容器非常相似，下面我们通过一个实例了解下chroot。

### chroot

什么和四chroot呢？下面是维基百科定义：

> chroot是在Unix和Linux系统的一个操作，针对正在运行的软件和它子进程，改变它外显的根目录。一个运行在这个环境下，经由chroot设置根目录的程序，他不能够对这个指定根目录之外的文件进行访问动作，不能读取，也不能变更它的内容。

通俗地说，chroot就是可以改变某进程的根目录，使这个程序不能夫纳顾问目录之外的其他目录，这个跟我们在一个容器中是很相似的。下面哦我们通过一个实例来演示下chroot

首先我们在当前目录下创建一个rootfs目录：

```bash
$ mkdir rootfs
```

这里这里为了方便演示，我使用现成的busybox镜像来创建一个系统，镜像的概念和组成后面会详细讲解，如果你没有DOcker基础可以把下面的操作命令成rootsfs下创建了一些目录和放置了一些二进制文件。

```bash
$ cd rootfs
$ docker export $(docker crete busybox) -o busybox.tar
$ tar -xf busybox.tar
```

执行完上面的命令后，在rootfs目录下初始化了一些目录，下面让我们通过一条命令来见证chroot的神奇之处。使用以下命令，可以启动一个sh进程，并且把`~/study/docker/rootfs`作为sh进程的根目录

```bash
$ sudo chroot ~/study/docker/rootfs /bin/sh
```

此时，我们的命令行窗口已处于上述命令启动的sh进程中。在当前sh命令行窗口下，我们使用ls命令查看以下当前进程，看是否真的与主机上的其他目录隔离开了。

```bash
/ # /bin/ls /
bin  busybox.tar  dev  etc  home  proc  root  sys  tmp  usr  var
```

这里可以看到当前进程的根目录已经变成了主机上的`~/study/docker/rootfs`目录。这样就实现了当前进程与主机的隔离。到此位置，一个目录隔离的容器就完成了。但是，此时还不能称之为一个容器，为什么？你可以在上一步（使用chroot启动命令行窗口）执行以下命令，查看如下路由信息：

```bash
/ # /bin/ip route
default via 192.168.0.1 dev enp3s0  src 192.168.0.103  metric 100 
172.17.0.0/16 dev docker0 scope link  src 172.17.0.1 
192.168.0.0/24 dev enp3s0 scope link  src 192.168.0.103  metric 100 
```

执行ip route命令后，你可以看到网络信息并没有隔离，实际上进程等信息此时也并未隔离。要先哦个实现一个完整的容器，我们还需要Linux的其他三项技术：Namespace、Cgroups和联合文件系统。

Docker是利用Linux的Namespace、Cgroups和联系文件系统三大机制来保证实现的，所以它的原理是使用Namespace做主机名、网络、PID等资源的隔离，使用Cgroups对进程或者进程组作资源（例如： CPU、内存等）的限制，联合文件系统用于镜像构建和容器运行环境。

后面会对这些技术进行详细讲解，这里简单解释一下它们的作用。

### Namespace

Namespace是Linux内和的一项功能，该功能对内和资源进行隔离，是的容器中的进程都可以在单独的命名空间中运行，并且只可以访问当前容器命名空间的资源。Namespace可以隔离进程ID、主机名、用户ID、文件名、网络访问和进程间通信等相关资源。

Docker 主要用到以下五种命名空间：

- pid namespace: 用户隔离进程ID。
- net namespace: 隔离网络接口，在虚拟的net namespace内用户可以拥有自己独立的IP、路由、端口等。
- mnt namespace: 文件系统挂载点隔离。
- ipc namespace: 信号量，消息队列和共享内存的隔离。
- uts namespace: 主机名和域名的隔离。

### Cgroups

Cgroups是一种Linux内核功能，可以限制和隔离进程的资源使用情况（CPU，内存，磁盘I/O、网络等）。在容器的实现中，Cgroups通常用来限制容器的CPU和内存等资源的使用。

### 联合文件系统

联合文件系统，又叫UnionFS，是一种通过创建文件层进程操作的文件系统，因此，联合文件系统非常快。Docker使用联合文件系统为容器提供构建层，使得容器可以实现写时复制以及镜像的分层和存储。常用的联合文件系统有AUFS、Overlay和Devicemapper等。

## 结语

容器技术从1979年chroot的首次问世便以湛露头角，但是是到了2013年，Docker的横空出世次阿是的容器技术的迅速发展，可见Docker对于容器技术的推动力和影响力。

> 另外，Docker还提供了工具和平台来管理容器的生命周期：
>
> 1. 使用容器开发应用程序及其支持组件。
> 2. 容器成为分发和测试你的应用程序的单元。
> 3. 可以将应用程序或协调服务器部署到生产环境中。无论您的生产环境是本地和数据中心，云提供商还是两者的混合，其工作原理都相同。

