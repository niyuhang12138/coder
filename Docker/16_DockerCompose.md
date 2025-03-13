# 如何使用Docker Compose解决开发环境的依赖？

到目前位置，我们所有的操作都是围绕着单个容器进行的，但当我们的业务越来越复杂的时候，需要多个容器相互配合，甚至需要多个主机组成容器集群才能满足我们的业务需求，这个时候就需要用到容器的编排工具了。因为容器的编排工具可以帮助我们批量的创建、调度和管理容器，帮助我们解决规模化容器的部署问题。

我们要介绍Docker三种常用的编排工具：Docker Compose、Docker Swarm和Kubernetes。了解了三这些编排工具，可以让你在不同的环境中选择最优的编排框架。

## Docker Compose的前世今生

Docker Compose的前身时Orchard公司开发的Fig，2014年Docker收购了这家公司，然后将Fig重命名为Docker Compose。现阶段Docker Compose是Docker官方的单机多容器管理系统，它本质是一个Python脚本，它通过解析用户编写的yaml文件，调用Docker API实现动态的创建和管理多个容器。

## 安装Docker Compose

Docker Compose可以安装在macOS、Windows、Linux系统中，其中在macOS和Windows系统下，Docker Compose都是随着Docker的安装一起安装好的。

### Linux系统下安装Docker Compose

**https://docs.docker.com/compose/install/linux/#install-the-plugin-manually**

```bash
$ DOCKER_CONFIG=${DOCKER_CONFIG:-$HOME/.docker}
$ mkdir -p $DOCKER_CONFIG/cli-plugins
$ curl -SL https://github.com/docker/compose/releases/download/v2.33.1/docker-compose-linux-x86_64 -o $DOCKER_CONFIG/cli-plugins/docker-compose
https://docs.docker.com/compose/install/linux/#install-the-plugin-manually
$ sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-compose
$ docker compose version
```

在使用Docker Compose之前，我们首先要先编写Docker Compose模板文件，因为Docker Compose运行的时候时根据Docker Compose模板文件中的定义来运行的。

## 编写Docker Compose模板文件

在使用Docker Compose启动容器的时候，Docker Compose会默认使用docker-compose.yml文件，docker-compose.yml文件的格式为yaml。

Docker Compose模板文件一个有三个版本： v1、v2、v3。目前最新的版本为v3,也是功能全面的一个版本，下面我们围绕着v3版本介绍一下如何编写Docker Compose文件。

Docker Compose模板文件一共有三部分：service（服务）、networks（网络）和volumes（数据卷）。

- service（服务）：服务定义了容器启动的各配置，就像我们执行`docker run`命令时传递的容器启动参数一样，指定了容器应该如何启动，例如容器的启动参数，容器的环境变量等，
- networks（网络）：网络定义了容器的网络配置，就像我们执行`docker network create`命令创建网络配置一样。
- volumes（数据卷）：数据卷定义了容器的卷配置，就像我们执行`docker volume create`命令创建数据卷一样。

一个典型的Docker Compose文件结构如下：

```yaml
version: "3"

services:

  nginx:

    ## ... 省略部分配置

networks:

  frontend:

  backend:

volumes:

  db-data:
```

### 编写Service配置

services下，首先需要定义服务名称，例如你这个服务是nginx服务，你可以定义service名称为nginx，格式如下：

```yaml
version: "3.8"
services:
	nginx:
```

服务名称定义完毕后，我们需要在服务名称的下一级定义当前服务的各项配置，使得我们的服务可以按照配置正常启动。

**build**： 用户构建Docker镜像，类似于`docker build`命令，build可以指定Dockerfile文件路径，然后根据Dockerfile命令来构建文件：

```yaml
build:
	## 构建执行的上下文目录
	context：
	## Dockerfile 名称
	dockerfile: Dockerfile-name
```

**cap_add、cap_drop**：指定容器可以使用到那些内核能力：

```yaml
cap_add:

  - NET_ADMIN

cap_drop:

  - SYS_ADMIN
```

**command**：用于覆盖容器的默认启动命令，它和Dockerfile中的CMD用法类似，也是有两种使用方式：

```yaml
command: sleep 3000
command: ["sleep", "3000"]
```

**container_name**：用于指定容器时容器的名称：

```yaml
container_name: nginx
```

**deoends_on**：用于指定服务间的依赖关系，这样就可以先启动被依赖的服务：

```yaml
version: "3.8"

services:

  my-web:

    build: .

    depends_on:

      - db

  db:

    image: mysql
```

**devices**：挂载主机的设备到容器中：

```makefile
devices:

  - "/dev/sba:/dev/sda"
```

**dns**：自定义容器中的DNS配置：

```yaml
dns:

  - 8.8.8.8

  - 114.114.114.114
```

**dns_search**：配置dns的搜索域：

```yaml
dns_search:

  - svc.cluster.com

  - svc1.cluster.com
```

**entrypoint**：覆盖容器的entrypoint命令：

```yaml
entrypoint: sleep 3000
entrypoint: ["sleep", "3000"]
```

**env_file**：指定容器的环境变量文件，启动的时候会把该文件中的环境变量注入容器中。

```bash
env_file:

  - ./dbs.env
```

env 文件的内容格式如下：

```ini
KEY_ENV=values
```

**environment**：指定容器启动时的环境变量：

```yaml
environment:
	KEY_ENV=values
```

**images**：指定容器镜像的地址：

```yaml
image: busybox:lastest
```

**pid：** 共享主机的进程命名空间，像在主机上直接启动进程一样，可以看到主机的进程信息。

```vbnet
pid: "host"
```

**ports：** 暴露端口信息，使用格式为 HOST:CONTAINER，前面填写要映射到主机上的端口，后面填写对应的容器内的端口。

```makefile
ports:

  - "1000"

  - "1000-1005"

  - "8080:8080"

  - "8888-8890:8888-8890"

  - "2222:22"

  - "127.0.0.1:9999:9999"

  - "127.0.0.1:3000-3005:3000-3005"

  - "6789:6789/udp"
```

**networks：** 这是服务要使用的网络名称，对应顶级的 networks 中的配置。

```markdown
services:

  my-service:

    networks:

     - hello-network

     - hello1-network
```

**volumes：** 不仅可以挂载主机数据卷到容器中，也可以直接挂载主机的目录到容器中，使用方式类似于使用`docker run`启动容器时添加 -v 参数。

```yaml
version: "3"

services:

  db:

    image: mysql:5.6

    volumes:

      - type: volume

        source: /var/lib/mysql

        target: /var/lib/mysql
```

volumes 除了上面介绍的长语法外，还支持短语法的书写方式，例如上面的写法可以精简为：

```javascript
version: "3"

services:

  db:

    image: mysql:5.6

    volumes:

      - /var/lib/mysql:/var/lib/mysql
```

### 编写Volume配置

如果你想在多个容器间共享数据卷，则需要在外部声明数据卷，然后在容器里声明使用数据卷。例如我想在两个服务间共享日志目录，则使用以下配置：

```yaml
version: "3"

services:

  my-service1:

    image: service:v1

    volumes:

      - type: volume

        source: logdata

        target: /var/log/mylog

  my-service2:

    image: service:v2

    volumes:

      - type: volume

        source: logdata

        target: /var/log/mylog

volumes:

  logdata:
```

### 编写Network配置

Docker Compose 文件顶级声明的 networks 允许你创建自定义的网络，类似于`docker network create`命令。

例如你想声明一个自定义 bridge 网络配置，并且在服务中使用它，使用格式如下：

```yaml
version: "3"

services:

  web:

    networks:

      mybridge: 

        ipv4_address: 172.16.1.11

networks:

  mybridge:

    driver: bridge

    ipam: 

      driver: default

      config:

        subnet: 172.16.1.0/24
```

## Docker Compose操作命令

我们可以使用`docker-compose -h`命令来查看docker-compose的用法，docker-compose的基本使用格式如下：

```bash
$ docker-compose [-f <arg>...] [options] [--] [COMMAND] [ARGS...]
```

其中options是docker-compose的参数，支持的参数和功能说明如下：

```
-f, --file FILE             指定 docker-compose 文件，默认为 docker-compose.yml

  -p, --project-name NAME     指定项目名称，默认使用当前目录名称作为项目名称

  --verbose                   输出调试信息

  --log-level LEVEL           日志级别 (DEBUG, INFO, WARNING, ERROR, CRITICAL)

  -v, --version               输出当前版本并退出

  -H, --host HOST             指定要连接的 Docker 地址

  --tls                       启用 TLS 认证

  --tlscacert CA_PATH         TLS CA 证书路径

  --tlscert CLIENT_CERT_PATH  TLS 公钥证书文件

  --tlskey TLS_KEY_PATH       TLS 私钥证书文件

  --tlsverify                 使用 TLS 校验对端

  --skip-hostname-check       不校验主机名

  --project-directory PATH    指定工作目录，默认是 Compose 文件所在路径。
```

COMMAND为docker-compose支持的命令：

```
 build              构建服务

  config             校验和查看 Compose 文件

  create             创建服务

  down               停止服务，并且删除相关资源

  events             实时监控容器的时间信息

  exec               在一个运行的容器中运行指定命令

  help               获取帮助

  images             列出镜像

  kill               杀死容器

  logs               查看容器输出

  pause              暂停容器

  port               打印容器端口所映射出的公共端口

  ps                 列出项目中的容器列表

  pull               拉取服务中的所有镜像

  push               推送服务中的所有镜像

  restart            重启服务

  rm                 删除项目中已经停止的容器

  run                在指定服务上运行一个命令

  scale              设置服务运行的容器个数

  start              启动服务

  stop               停止服务

  top                限制服务中正在运行中的进程信息

  unpause            恢复暂停的容器

  up                 创建并且启动服务

  version            打印版本信息并退出
```

## 使用Docker Compose管理WordPress

### 启动wordPress

第一个创建项目目录：

```bash
$ mkdir tmp/wordpress
```

进入工作目录：

```bash
cd tmp/wordpress
```

创建docker-compose.yml文件：

```bash
$ touch docker-compose.yml
```

然后写入以下内容：

```yaml
version: '3'
services:
  mysql:
    image: mysql:latest
    volumes:
      - mysql_data:/var/lib/mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: mywordpress
      MYSQL_USER: niyuhang
      MYSQL_PASSWORD: nyh196511
  wordpress:
    depends_on:
      - mysql
    image: wordpress:latest
    ports:
      - "8000:80"
    restart: always
    environment:
      WORDPRESS_DB_HOST: mysql:3306
      WORDPRESS_DB_USER: niyuhang
      WORDPRESS_DB_PASSWORD: nyh196511
      WORDPRESS_DB_NAME: mywordpress
volumes:
  mysql_data: {}
```

启动MySQL数据库和WordPress服务：

```bash
$ docker compose up -d
please remove it to avoid potential confusion 
[+] Running 4/4
 ✔ Network wordpress_default        Created                                0.1s 
 ✔ Volume "wordpress_mysql_data"    Cre...                                 0.0s 
 ✔ Container wordpress-mysql-1      Start...                               0.5s 
 ✔ Container wordpress-wordpress-1  S...                                   0.6s 
```

停止：

```bash
$ docker compose stop
```

## 结语

Docker Compose是一个用来定义复杂应用的单机编排工具，通常用于服务以以来关系复杂的开发和测试环境。