# Docker 容器操作

## 容器（Container）是什么？

容器是基于镜像创建的可运行实例，并且单独存在，一个镜像可以创建出多个容器。运行容器化环境时，实际上是在容器内部创建该文件系统的读写副本。浙江添加一个容器层，该层允许修改镜像的整个副本。

![image-20250312154920684](./assets/image-20250312154920684.png)

## 容器的生命周期

容器的生命周期是容器可能处于的状态，容器的生命周期分为五种：

1. created：初始状态
2. running：运行状态
3. stopped：停止状态
4. paused：暂停状态
5. deleted：删除状态

各生命周期之前的转换关系如图所示：

![image-20250312155228546](./assets/image-20250312155228546.png)

通过`docker create`命令生成的容器状态为初建状态，初建状态通过`docker start`命令可以转换为运行状态，运行状态的容器可以通过`docker stop`命令转换为停止状态，处于停止状态的容器可以日通过`docker start`转换为运行状态，运行状态的容器也可以通过`docker pause`命令转换为暂停状态，处于暂停状态的容器可以通过`docker unpause`转换为运行状态。处于初建状态、运行状态、停止状态、暂停状态的容器都可以直接删除。

## 容器的操作

容器的操作可以分为五个步骤：创建并启动容器、终止容器、进入容器、删除容器、导入和导出容器

### 创建并启动容器

容器十分轻量，用户可以随时创建和删除它。我们可以使用`docker create`命令来创建容器：

```bash
$ docker create -it --name=<name> <Image>
```

如果使用`docker create`命令创建的容器处于停止状态，我们可以使用`docker start`命令来启动它：

```bash
$ docker start <name>
$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
d6f3d364fad3        busybox             "sh"                16 seconds ago      Up 8 seconds                            <name>
```

这时候我们可以看到容器已经处于启动状态了。容器启动有两种方式：

1. 使用`docker start`命令基于已经创建好的容器直接启动。
2. 使用`docker run`命令直接基于镜像新建一个容器并启动，相当于先执行`docker create`命令从镜像创建容器，然后在执行`docker start`命令启动容器。

使用`docker run`的命令如下：

```bash
$ docker urn -it --name=<name> <Image>
```

当使用`docker run`创建并启动容器时，Docker后台执行的流程为：

- Docker会检查本地是否存在busybox镜像，如果镜像不存在则从Docker Hub拉取busybox镜像；
- 使用busybox镜像创建并启动一个容器；
- 分配文件系统，并且在镜像只读层外创建一个读写层；
- 从Docker IP池中分配一个IP给容器；
- 执行用户的启动命令运行镜像

上述命令中，-t参数的作用是分配一个伪终端，-i参数则可以终端的STDIN打开，同时使用-it参数可以让我们进入交互模式。在交互模式下，用户可以通过所创建的终端来输入命令，例如：

```bash
$ ps aux
PID   USER     TIME  COMMAND
    1 root      0:00 sh
    6 root      0:00 ps aux
```

我们可以看到容器的1号进程为sh命令，在容器内部并不能看到主机上的进程信息，以为内容器内部和主机是完全隔离的。同时用于sh是1号进程，以为iezhe如果通过exit推出sh，那么容器也会推出。所以对于容器来说，沙溪容器中的主进程，则容器也会被杀死。

### 终止容器

容器启动后，如果我们想停止运行中的容器，可以使用`docker stop`命令。命令格式为`docker stop [-t]-rime=[=10]。该命令首先会向运行中的容器发送SIGTERM信号，如果容器内1号县城接受并能够红醋栗SIGTERM，则等待1号县城处理完毕后退出，如果等待一段时间后，容器仍然没有推出，则会发送SIGKILL强制终止容器。

```bash
$ docker stop busybox
busybox
```

如果你要查看停止状态的容器信息，你可以使用docker ps -a命令：

```bash
$ docker ps -a
CONTAINERID       IMAGE      COMMAND            CREATED             STATUS     PORTS         NAMES
28d477d3737a        busybox             "sh"                26 minutes ago      Exited (137) About a minute ago                       busybox
```

处于终止状态的容器也可以通过`docker start`命令来重新启动。

```bash
$ docker start busybox
busybox
$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
28d477d3737a        busybox             "sh"                30 minutes ago      Up 25 seconds                           busybox
```

此外，`docker resatrt`命令会将一个运行好的容器停止，并且重新启动它。

```bash
$ docker restart busybox
busybox
$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
28d477d3737a        busybox             "sh"                32 minutes ago      Up 3 seconds                            busybox
```

### 进入容器

处于运行状态的东起可以通过`docker attach`、`docker exec`、`nsenter`等多种方式进入容器。

#### 使用`docker attach`命令进入容器

使用`docker attach`，进入我们上一步创建好的容器：

```bash
$ docker attach busybox
/ # ps aux
PID   USER     TIME  COMMAND
    1 root      0:00 sh
    7 root      0:00 ps aux
/ #
```

注意：当我们同时使用`docker attach`命令同时在多个终端运行时，所有的终端窗口将同步显示相同内容，当某个命令好窗口的命令阻塞时，其他命令行窗口同样也无法操作。由于`docker attach`命令不够灵活，因为我们一般不会使用`docker attach`进入容器。

#### 使用`docker exec`命令进入容器

Docker从1.3版本开始，提供了一个更加方便的进入容器的命令`docker exec`，我们可以通过`docker exec it CONTAINER`的方式进入到一个已经运行中的容器：

```bash
$ docker exec -it busybox sh
/ # ps aux
PID   USER     TIME  COMMAND
    1 root      0:00 sh
    7 root      0:00 sh
   12 root      0:00 ps aux
```

我们进入容器后，可以看到有两个`sh`进程，这和四因为以`exec`的方式进入容器，会单独启动一个sh进程，每个窗口都是独立且互补打扰的，也是使用最多的一种方式。

### 删除容器

我们已经掌握用Docker命令创建、启动和终止容器。那如何删除处于终止状态或者运行中的容器呢？

删除容器命令的使用方式如下：`docker rm [OPTIONS] CONTAINER [CONTAINER..]`

```bash
$ docker rm busybox
```

如果要删除正在运行中的容器，必须添加-f（或者--force）参数，Docker会发送SIGKILL信号强制终止正在运行的容器。

```bash
$ docker rm -f busybox
```

### 导出导入容器

#### 导出容器

我们可以使用`docker exprot CONTAINER`命令导出一个容器到文件，不管此时该容器是否处于运行中的状态。导出容器前我们县进入容器，创建一个我呢间，过程如下：

```bash
$ docker exec -it busybox sh
$ cd /tmp && touch test
```

然后在执行导出命令

```bash 
$ docker export busybox > busybox.tar
```

执行以上命令会在当前文件夹下生成busybox.tar文件，我们可以将该文件拷贝到其他机器上，通过导入命令实现容器的迁移。

#### 导入容器

通过`docker export`命令导出的文件，可以使用`docker import`命令导入，执行完`docker import`后会变为本地镜像，最后在使用`docker run`命令启动该镜像，这样我们就实现了容器的迁移。

导入容器的命令格式为`docker import [OPTIONS] file|URL [REPOSITORY[:TAG]]`。接下来我们一步步将上一步导出的镜像文件导入到其他机器的Docker中并启动它。

首先，使用`docker import`命令导入上一步导出的容器

```bash
$ docker import busybox.tar busybox:test
```

此时，busybox.tar被导入成为新的镜像，镜像名称为busybox:test。下面，我们使用`docker run`命令启动并进入容器，查看上一步创建的临时文件

```bash
$ docker run -it busybox:test sh
/ # /ls /tmp/
test
```

可以看到我们之前在/tmp目录下创建的test文件也被迁移过来了。这样我们就通过`docker export`和`docker import`命令配合实现了容器的迁移。

## 结语

容器和镜像的区别：镜像包含了容器运行所需要的文件系统结构和内容，是静态的只读文件，而容器则是在镜像的只读层上创建了可写层，并且容器中的进程属于运行状态，容器是真正的应用载体。