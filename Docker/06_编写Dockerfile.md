# 最佳实践：如何在生产中编写最优Dockerfile？

生产实践中一定优先使用Dockerfile的方式构建镜像。因为使用Dockerfile构建镜像可以带来很多好处：

- 易于版本管理，Dockerfile本身是一个文本文件，方便存放在代码仓库做版本管理，可以很方便的找到各个版本之间的变更历史；
- 过程可追溯，Dockerfile的每一行的指令代表一个镜像层，根据Dockerfile的内容即可很明确的查看镜像的完整构建过程；
- 评比构建环境异构，使用Dockerfile构建镜像无需考虑构建环境，基于相同Dockerfile无论在哪里运行，构建结构都是一致的。

虽然有这么多好处，但是如果你Dockerfile使用不当也会引发很多问题。比如镜像构建时间过长，甚至镜像构建和失败；镜像层数过多，导致镜像文件过大。所以我们先来考虑如何在生产环境中编写最优的Dockerfile。

## Dockerfile的书写原则

遵循一下Dockefile书写原则，不仅可以使得我们的Dockerfile简洁明了，让协作者清楚的了解镜像的完整构建流程，还可以帮助我们减少镜像的体积，加快镜像构建的速度和分发速度。

### 单一职责

由于容器本质是进程，一个容器代表一个进程，因此不同功能的应用应用尽量分为不同的容器，每个单一的容器只负责单一业务进程。

### 提供注释信息

Dockefile也是一种代码，我们应该保持良好的代码编写习惯，晦涩难懂的代码尽量添加注释，让协作者可以一目了然的知道每一行代码的作用，并且方便拓展和使用。

### 提供容器最小化

应该避免安装无用的软件包，比如在一个nginx镜像中，我并不需要安装vim、gcc等开发编译工具。这样不仅可以加快容器构建速度，而且可以避免镜像体积过大。

### 合理选择基础镜像

容器的核心是应用，因此只要基础镜像能够满足应用的运行环境即可。例如一个Java类型的应用运行时只需要JRE，并不需要JDK，因此我们的基础镜像只需要安装JRE环境即可。

### 使用.dockerignore文件

在使用git的时候，我们可以使用.gitignore文件忽略一些不徐亚的做版本管理的文件。同理，使用.dockerignore文件允许我们在构建时，忽略一些不需要参与构建的文件，从而提升构建效率。.dockerignore的定义类似于.gitignore

.dockerignore的本质是文本文件，Docker构建时可以使用同换行符来解析文件定义，每一行可以据略一些文件或者文件夹，具体的使用方式如下：

| 规则       | 含义                                                         |
| :--------- | :----------------------------------------------------------- |
| #          | # 开头的表示注释，# 后面所有内容将会被忽略                   |
| */tmp*     | 匹配当前目录下任何以 tmp 开头的文件或者文件夹                |
| *.md       | 匹配以 .md 为后缀的任意文件                                  |
| tem?       | 匹配以 tem 开头并且以任意字符结尾的文件，？代表任意一个字符  |
| !README.md | ! 表示排除忽略。 例如 .dockerignore 定义如下： *.md !README.md 表示除了 README.md 文件外所有以 .md 结尾的文件。 |

### 尽量使用构建缓存

Docker构建过程中，每一条Dockerfile指令都会提交为一个镜像层，下一条指令都是基于上一条指令构建的。如果构建时发现要构建的父镜像层已经存在，并且下一条指令使用了相同的指令，即可命中构建缓存。

Docker构建时判断是否需要使用缓存的规则如下：

- 从当前构建层开始，比较所有的子镜像，检查所有的构建指令是否与当前完全一致，如果不一致，则不使用缓存。
- 一般情况下，只需要比较构建指令即可判断是否需要缓存，但是有些指令除外（例如ADD和COPY）。
- 对于ADD和COPY指令不仅要检验命令是否一致，还要为即将拷贝到容器的文件计算校验和（根据文件内容计算出的一个数值，如果两个文件计算的数值一致，表示两个文件内容一致），命令和校验和完全一致，才认为命中缓存。

因为，基于Docker构建时的缓存特性，我们可以把不轻易改变的指令放到Dockerfile的前面（例如安装软件包），而可能经常发生改变的指令放在Dockerfile末尾（例如编译应用程序）。

例如，我们想要想要定义一些环境变量并且安装一些软件，可以按照如下顺序编写Dockerfile：

```dockerfile
FROM centos:7
# 设置环境变量指令放前面
ENV PATH /usr/local/bin:$PATH
# 安装软件指令放前面
RUN yum install -y make
# 把业务软件的配置,版本等经常变动的步骤放最后
...
```

按照上面原则编写的Dockerfile在构建镜像时候，前面步骤命中缓存的概率会大大增加，可以大大缩短镜像构建时间。

### 正确设置时区

我们从Docker Hub拉取官方操作系统镜像大多数都是UTC时间（世界标准时间）。如果你想要在容器中使用中国区标准时间，请根据使用的操作系统修改相应的时区信息，下面我介绍几种常用操作系统的修改方式“

- **Ubuntu 和Debian 系统**

Ubuntu 和Debian 系统可以向 Dockerfile 中添加以下指令：

```bash
RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
RUN echo "Asia/Shanghai" >> /etc/timezone
```

- **CentOS系统**

CentOS 系统则向 Dockerfile 中添加以下指令：

```bash
RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```

### 使用国内软件源加速构建速度

由于我们常用的官方操作系统镜像基本都是国外，软件服务器大部分也在国外，所以我们构建镜像时想要安装一些软件包可能会非常慢。

### 最小化镜像层数

在构建镜像时尽可能的减少Dockerfile指令行数，例如我们要在Ubuntu系统中安装make和net-tools这两个软件包，应该在Dockerfile中使用以下指令：

```go
RUN apt install make net-tools
```

而不应该写成这样：

```go
RUN apt install make
RUN apt install net-tools
```

## Docker指令书写建议

下面是我们常用的一些指令，这些指令对于更接触Docker的人来说会非常容易出错。

### RUN

RUN指令在构建时将会生成一个新的镜像层并执行RUN指令后面的内容。

使用RUN指令时英国尽量遵循以下原则：

- 当RUN指令后面跟的内容比较复杂的时候，建议使用反斜杠结尾并且换行。
- RUN指令后面的内容尽量按照字母顺序排序，提高可读性。

例如，我们在官方的Ubuntu镜像下安装一些软件，一个建议的Dock指令如下：

```dockerfile
FROM ubuntu: 24.04
RUN yum install automake \
                curl \
                python \
                vim
```

### CMD和ENTRYPOINT

CMD和ENTRYPOINT指令都是容器运行的命令入口，这两个指令使用中有很多相似的地方，但是也有一些区别。

这两个指令的相同之处，CMD和ENTRYPOINT的基本使用格式分为两种。

- 第一种为`CMD/ENTRYPOINT["command", "param"]`。这种格式是使用Linux的exec实现的，一般成为exec模式，这种书写格式为CMD/ENTRYPOINT后面跟json数组，也是Docker推荐使用格式。
- 另一种格式为CMD/ENTRYPOINT command param，这种格式是基于shell实现的，通常成为shell模式。当使用shell模式的时候，Docker会以`/bin/sh -c command`的方式执行命令。

这两个指令的区别：

- Dockerfile中如果使用ENTRYPOINT指令，启动Docker容器时需要使用--entrypoint参数才能覆盖Dockerfile中的ENTRYPOINT指令，而是用CMD设置的命令可以被`docker run`后面的参数直接覆盖。
- ENTRYPOINT指令可以结合CMD指令使用，也可以单独使用，而CMD指令只能单独使用。

那什么时候应该使用ENTRYPOINT，什么使用应该使用CMD呢？

如果哦你希望你的镜像足够灵活，推荐使用CMD指令。如果你的镜像支持性单一的具体程序，并且不希望用户在执行`docker run`的时候覆盖默认程序，建议使用ENTRYPOINT。

无论是用CMD还是ENTRYPOINT，都尽量使用exec模式。

### ADD和COPY

ADD和COPY指令功能类似，都是从外部往容器内添加文件。但是COPY指令只支持基本的文件和文件拷贝功能，ADD则支持更多文件来源类型，比如自动提取tar包，并且可以支持源文件为URL格式。

那么在日常应用中，我们应该使用那个命令向容器里添加文件呢？你可能在想，既然ADD指令支持的功能更多，当然应该使用ADD指令了。然而事实恰恰相反，推荐使用COPY指令，因为COPY指令更加透明，金支持本地文件向容器拷贝，而且使用COPY指令可以更好的利用构建缓存，有效的建校镜像体积。

当你想要使用ADD向容器中添加URL文件时，清尽量使用其他方式代替他：

```dockerfile
ADD http://pyropus.ca/software/memtester/old-versions/memtester-4.3.0.tar.gz /tmp/
RUN tar -xvf /tmp/memtester-4.3.0.tar.gz -C /tmp
RUN make -C /tmp/memtester-4.3.0 && make -C /tmp/memtester-4.3.0 install
```

下面是推荐写法：

```dockerfile
RUN wget -O /tmp/memtester-4.3.0.tar.gz http://pyropus.ca/software/memtester/old-versions/memtester-4.3.0.tar.gz \
&& tar -xvf /tmp/memtester-4.3.0.tar.gz -C /tmp \
&& make -C /tmp/memtester-4.3.0 && make -C /tmp/memtester-4.3.0 install
```

### WORKDIR

为了使构建过程更加清晰明了，推荐使用WORKDIR来指定容器的工作路由