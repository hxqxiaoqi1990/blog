---
title: dockerfile基本使用
tags: [docker]
categories: kubernetes
date: 2019-06-14 18:27:43
---


# dockerfile作用
<font color=DarkTurquoise>**dockerfile**</font> 是由一系列命令和参数构成的脚本，这些命令应用于基础镜像并最终创建一个新的镜像。它们简化了从头到尾的流程并极大的简化了部署工作。Dockerfile从FROM命令开始，紧接着跟随者各种方法，命令和参数。其产出为一个新的可以用于创建容器的镜像。

简单来说，dockerfile就是用来快速定制和生成自己的镜像。

# dockerfile基本语法

## FROM
指定基础镜像
所谓定制镜像，那一定是以一个镜像为基础，在其上进行定制。而 FROM 就是指定基础镜像，因此一个 Dockerfile 中 FROM 是必备的指令，并且必须是第一条指令。

除了选择现有镜像为基础镜像外，Docker 还存在一个特殊的镜像，名为 scratch 。这个镜像是虚拟的概念，并不实际存在，它表示一个空白的镜像。

`FROM scratch`

如果你以 scratch 为基础镜像的话，意味着你不以任何镜像为基础，接下来所写的指令将作为镜像第一层开始存在。

不以任何系统为基础，直接将可执行文件复制进镜像的做法并不罕见，比如 swarm 、 coreos/etcd 。对于 Linux 下静态编译的程序来说，并不需要有操作系统提供运行时支持，所需的一切库都已经在可执行文件里了，因此直接 FROM scratch 会让镜像体积更加小巧。使用 Go 语言 开发的应用很多会使用这种方式来制作镜像，这也是为什么有人认为 Go是特别适合容器微服务架构的语言的原因之一。

## RUN 
RUN 指令是用来执行命令行命令的。由于命令行的强大能力， RUN 指令在定制镜像时是最常用的指令之一。

在指令格式上，一般推荐使用 exec 格式，这类格式在解析时会被解析为 JSON 数组，因此一定要使用双引号 " ，而不要使用单引号。

其格式有两种：
shell 格式：

``` bash
RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
```

exec 格式： 

``` bash
RUN ["可执行文件", "参数1", "参数2"]
```

注意：Dockerfile 中每一个指令都会建立一层，最大限制为127层，有些类似的指令可以使用`&&`连接，减少层数。

## COPY 
复制文件

格式：

``` bash
COPY <源路径> <目标路径>

COPY ["<源路径1>","<目标路径>"]
```

注意：

 1. 源路径不能使用绝对路径
 2. 复制到容器的文件和状态会被保留
 3. 支持通配符

## ADD 
更高级的复制文件

格式与copy一致

如果 <源路径> 为一个 tar 压缩文件的话，压缩格式为 gzip , bzip2 以及 xz 的情况下， ADD 指令将会自动解压缩这个压缩文件到 <目标路径> 去。
另外需要注意的是， ADD 指令会令镜像构建缓存失效，从而可能会令镜像构建变得比较缓慢。

## CMD 
容器启动命令

CMD 指令的格式和 RUN 相似。

Docker 不是虚拟机，容器中的应用都应该以前台执行，而不是像虚拟机、物理机里面那样，容器内没有后台服务的概念，所有容器必须有一个前台进程，否则无法启动容器。

## ENTRYPOINT
入口点

ENTRYPOINT 的格式和 RUN 指令格式一样。

ENTRYPOINT 的目的和 CMD 一样，都是在指定容器启动程序及参数，一个dockerfile中只能有一个ENTRYPOINT 层。


`CMD`可以指定参数传递到`ENTRYPOINT`，例：

``` bash
FROM alpine:3.4 ... RUN addgroup -S redis && adduser -S -G redis redis ... 
ENTRYPOINT ["docker-entrypoint.sh"] 
EXPOSE 6379 
CMD [ "redis-server" ]
```

``` bash
#!/bin/sh ... 
#allow the container to be started with `--user` 
if [ "$1" = 'redis-server' -a "$(id -u)" = '0' ]; then 
	chown -R redis .
	exec su-exec redis "$0" "$@" 
fi 
exec "$@"
```
该脚本的内容就是根据 `CMD` 的内容来判断，如果是 `redis-server`的话，则切换到 redis 用户身份启动服务器，否则依旧使用 root 身份执行。

## ENV 
设置环境变量

格式有两种：

``` bash
ENV <key> <value>
ENV <key1>=<value1> <key2>=<value2>...
```

下列指令可以支持环境变量：
ADD 、 COPY 、 ENV 、 EXPOSE 、 LABEL 、 USER 、 WORKDIR 、 VOLUME 、 STOPSIGNAL 、 ONBUILD 。

## ARG 
构建参数

``` bash
ARG <参数名>[=<默认值>]
```

Dockerfile 中的 ARG 指令是定义参数名称，以及定义其默认值。该默认值可以在构建命令 docker build 中用 --build-arg <参数名>=<值> 来覆盖。

在 1.13 之前的版本，要求 –build-arg 中的参数名，必须在 Dockerfile 中用 ARG 定义过了，换句话说，就是 –build-arg 指定的参数，必须在 Dockerfile 中使用了。如果对应参数没有被使用，则会报错退出构建。从 1.13 开始，这种严格的限制被放开，不再报错退出，而是显示警告信息，并继续构建。这对于使用 CI 系统，用同样的构建流程构建不同的 Dockerfile 的时候比较有帮助，避免构建命令必须根据每个 Dockerfile 的内容修改。

## VOLUME 
定义匿名卷

``` bash
VOLUME ["<路径1>", "<路径2>"...]
VOLUME <路径>
```
容器运行时应该尽量保持容器存储层不发生写操作，对于数据库类需要保存动态数据的应用，其数据库文件应该保存于卷(volume)中。为了防止运行时用户忘记将动态文件所保存目录挂载为卷，在 Dockerfile 中，我们可以事先指定某些目录挂载为匿名卷，这样在运行时如果用户不指定挂载，其应用也可以正常运行，不会向容器存储层写入大量数据。

``` bash
VOLUME /data
```

这里的 /data 目录就会在运行时自动挂载为匿名卷，任何向 /data 中写入的信息都不会记录进容器存储层，从而保证了容器存储层的无状态化。当然，运行时可以覆盖这个挂载设置。比如：

``` bash
docker run -d -v mydata:/data xxxx
```

在这行命令中，就使用了 mydata 这个命名卷挂载到了 /data 这个位置，替代了 Dockerfile 中定义的匿名卷的挂载配置。

## EXPOSE 
声明端口

``` bash
EXPOSE <端口1> <端口2>
```

EXPOSE 指令是声明运行时容器提供服务端口，这只是一个声明，在运行时并不会因为这个声明应用就会开启这个端口的服务。在 Dockerfile 中写入这样的声明有两个好处，一个是帮助镜像使用者理解这个镜像服务的守护端口，以方便配置映射；另一个用处则是在运行时使用随机端口映射时，也就是 docker run -P 时，会自动随机映射 EXPOSE 的端口。

要将 EXPOSE 和在运行时使用 -p <宿主端口>:<容器端口> 区分开来。 -p ，是映射宿主端口和容器端口，换句话说，就是将容器的对应端口服务公开给外界访问，而 EXPOSE 仅仅是声明容器打算使用什么端口而已，并不会自动在宿主进行端口映射。

## WORKDIR
指定工作目录

``` bash
WORKDIR <工作目录路径>
```

使用 WORKDIR 指令可以来指定工作目录（或者称为当前目录），以后各层的当前目录就被改为指定的目录，如该目录不存在， WORKDIR 会帮你建立目录。

## USER
指定当前用户

``` bash
USER <用户名>
```

USER 指令和 WORKDIR 相似，都是改变环境状态并影响以后的层。 WORKDIR 是改变工作目录， USER 则是改变之后层的执行 RUN , CMD 以及 ENTRYPOINT 这类命令的身份。当然，和 WORKDIR 一样， USER 只是帮助你切换到指定用户而已，这个用户必须是事先建立好的，否则无法切换。

## HEALTHCHECK 
健康检查

``` bash
HEALTHCHECK [选项] CMD <命令> ：设置检查容器健康状况的命令
```

当在一个镜像指定了 HEALTHCHECK 指令后，用其启动容器，初始状态会为 starting ，在 HEALTHCHECK 指令检查成功后变为 healthy ，如果连续一定次数失败，则会变为 unhealthy 。

HEALTHCHECK 支持下列选项：

 1. interval=<间隔> ：两次健康检查的间隔，默认为 30 秒；
 2. timeout=<时长> ：健康检查命令运行超时时间，如果超过这个时间，本次健康检查就被视为失败，默认 30 秒；
 3. retries=<次数> ：当连续失败指定次数后，则将容器状态视为 unhealthy ，默认 3 次。

和 CMD , ENTRYPOINT 一样， HEALTHCHECK 只可以出现一次，如果写了多个，只有最后一个生效。

例：

``` bash
FROM nginx
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*
HEALTHCHECK --interval=5s --timeout=3s \
    CMD curl -fs http://localhost/ || exit 1
```
使用 `curl -fs http://localhost/ || exit 1`作为健康检查命令。
当运行该镜像后，可以通过 `docker container ls`

HEALTHCHECK有三种状态：

 1. starting：当运行该镜像后的最初状态
 2. healthy：容器运行稳定后的健康状态
 3. unhealthy：健康检查连续失败超过了重试次数

为了帮助排障，健康检查命令的输出（包括 stdout 以及 stderr ）都会被存储于健康状态里，可以用 docker inspect 来查看。