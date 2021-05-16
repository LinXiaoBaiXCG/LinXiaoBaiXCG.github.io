---
title: Dockerfile详解
date: 2020-05-20 20:00:00
tags: [工具]
categories: 编程
---

# Dockerfile？

Dockerfile 是一个用来构建镜像的文本文件，文本内容包含了一条条构建镜像所需的指令和说明。

# Dockerfile的指令

Dockerfile的十三个基本指令：

| 类型               | 命令                                                        |
| ------------------ | ----------------------------------------------------------- |
| 基础镜像信息       | FROM                                                        |
| 维护者信息         | MAINTAINER                                                  |
| 镜像操作指令       | RUN、COPY、ADD、EXPOSE、WORKDIR、ONBUILD、USER、VOLUME、ENV |
| 容器启动时执行指令 | CMD、ENTRYPOINT                                             |

## FROM：指定基础镜像

所谓定制镜像，那一定是以一个镜像为基础，在其上进行定制。基础镜像是必须指定的，并且必须是第一条指令。

如：指定centos作为基础镜像

```
FROM centos
```

## MAINTAINER：指定作者

用来指定dockerfile的作者名称和邮箱，主要作用是为了标识软件的所有者是谁。
语法：

```
MAINTAINER <name> <email>
```

如：

```
MAINTAINER autor_xxx xxx@qq.com
```

## RUN：执行命令

RUN指令在新镜像内部执行的命令，如：执行某些动作、安装系统软件、配置系统信息之类，

格式如下两种：

1. shell格式：RUN< command > ，就像直接在命令行中输入的命令一样。

2. exec格式：RUN ["可执行文件", "参数1", "参数2"]

如在新镜像中用yum方式安装nginx：

```
RUN ["yum","install","nginx"]
```

注：多行命令不要写多个RUN，原因是Dockerfile中每一个指令都会建立一层.多少个RUN就构建了多少层镜像，会造成镜像的臃肿、多层，不仅仅增加了构件部署的时间，还容易出错,RUN书写时的换行符是\

## COPY：复制文件

COPY命令用于将宿主机器上的的文件复制到镜像内，如果目的位置不存在，Docker会自动创建。但宿主机器用要复制的目录必须是和Dockerfile文件统计目录下。

格式：

```
COPY [--chown=<user>:<group>] <源路径>... <目标路径>
COPY [--chown=<user>:<group>] ["<源路径1>",... "<目标路径>"]
```

如把宿主机中的package.json文件复制到容器中/usr/src/app/目录下：

```
COPY package.json /usr/src/app/
```

## ADD

ADD 指令和 COPY 的使用格式一致（同样需求下，官方推荐使用 COPY）。功能也类似，不同之处如下：

- ADD 的优点：在执行 <源文件> 为 tar 压缩文件的话，压缩格式为 gzip, bzip2 以及 xz 的情况下，会自动复制并解压到 <目标路径>。
- ADD 的缺点：在不解压的前提下，无法复制 tar 压缩文件。会令镜像构建缓存失效，从而可能会令镜像构建变得比较缓慢。具体是否使用，可以根据是否需要自动解压来决定。

## EXPOSE：暴露端口

EXPOSE命名适用于设置容器对外映射的容器端口号，如tomcat容器内使用的端口8081，则用EXPOSE命令可以告诉外界该容器的8081端口对外，在构建镜像时
用docker run -p可以设置暴露的端口对宿主机器端口的映射。

语法：

```
EXPOSE <端口1> [<端口2>...]
```

如：

```
EXPOSE 8081
```

EXPOSE 8081 其实等价于 docker run -p 8081 当需要把8081端口映射到宿主机中的某个端口（如8888）以便外界访问时，则可以用docker run -p 8888:8081

## WORKDIR：配置工作目录

WORKDIR命令是为RUN、CMD、ENTRYPOINT指令配置工作目录。其效果类似于Linux命名中的cd命令，用于目录的切换，但是和cd不一样的是：如果切换到的目录不存在，WORKDIR会为此创建目录。

语法:

```
WORKDIR path
```

## ONBUILD

ONBUILD用于配置当前所创建的镜像作为其它新创建镜像的基础镜像时，所执行的操作指令。
意思就是：这个镜像创建后，如果其它镜像以这个镜像为基础，会先执行这个镜像的ONBUILD命令
格式：

```
ONBUILD [INSTRUCTION]
```

## USER

用于指定执行后续命令的用户和用户组，这边只是切换后续命令执行的用户（用户和用户组必须提前已经存在）。

格式：

```
USER <用户名>[:<用户组>]
```

### VOLUME

定义匿名数据卷。在启动容器时忘记挂载数据卷，会自动挂载到匿名卷。

作用：

- 避免重要的数据，因容器重启而丢失，这是非常致命的。
- 避免容器不断变大。

格式：

```
VOLUME ["<路径1>", "<路径2>"...]
VOLUME <路径>
```

在启动容器 docker run 的时候，我们可以通过 -v 参数修改挂载点。

## ENV

设置环境变量，定义了环境变量，那么在后续的指令中，就可以使用这个环境变量。

格式：

```
ENV <key> <value>
ENV <key1>=<value1> <key2>=<value2>...
```

以下示例设置 NODE_VERSION = 7.2.0 ， 在后续的指令中可以通过 $NODE_VERSION 引用：

```
ENV NODE_VERSION 7.2.0

RUN curl -SLO "https://nodejs.org/dist/v$NODE_VERSION/node-v$NODE_VERSION-linux-x64.tar.xz" \
  && curl -SLO "https://nodejs.org/dist/v$NODE_VERSION/SHASUMS256.txt.asc"
```

## CMD

类似于 RUN 指令，用于运行程序，但二者运行的时间点不同:

- CMD 在docker run 时运行。
- RUN 是在 docker build。

**作用**：为启动的容器指定默认要运行的程序，程序运行结束，容器也就结束。CMD 指令指定的程序可被 docker run 命令行参数中指定要运行的程序所覆盖。

**注意**：如果 Dockerfile 中如果存在多个 CMD 指令，仅最后一个生效。

格式：

```
CMD <shell 命令> 
CMD ["<可执行文件或命令>","<param1>","<param2>",...] 
CMD ["<param1>","<param2>",...]  # 该写法是为 ENTRYPOINT 指令指定的程序提供默认参数
```

推荐使用第二种格式，执行过程比较明确。第一种格式实际上在运行的过程中也会自动转换成第二种格式运行，并且默认可执行文件是 sh。

## ENTRYPOINT：容器启动执行命名

ENTRYPOINT的作用和用法和CMD一模一样，但是ENTRYPOINT有和CMD有2处不一样：

1. CMD的命令会被docker run的命令覆盖而ENTRYPOINT不会
2. CMD和ENTRYPOINT都存在时，CMD的指令变成了ENTRYPOINT的参数，并且此CMD提供的参数会被 docker run 后面的命令覆盖

示例：

假设已通过 Dockerfile 构建了 nginx:test 镜像：

```
FROM nginx

ENTRYPOINT ["nginx", "-c"] # 定参
CMD ["/etc/nginx/nginx.conf"] # 变参 
```

1、不传参运行

```
$ docker run  nginx:test
```

容器内会默认运行以下命令，启动主进程。

```
nginx -c /etc/nginx/nginx.conf
```

2、传参运行

```
$ docker run  nginx:test -c /etc/nginx/new.conf
```

容器内会默认运行以下命令，启动主进程(/etc/nginx/new.conf:假设容器内已有此文件)

```
nginx -c /etc/nginx/new.conf
```

# 编写Dockerfile

示例：

```
#在centos上安装nginx
FROM centos
#标明著作人的名称和邮箱
MAINTAINER jiabuli 649917837@qq.com
#测试一下网络环境
RUN ping -c 1 www.baidu.com
#安装nginx必要的一些软件
RUN yum -y install gcc make pcre-devel zlib-devel tar zlib
#把nginx安装包复制到/usr/src/目录下
ADD nginx-1.15.8.tar.gz /usr/src/
#切换到/usr/src/nginx-1.15.8编译并且安装nginx
RUN cd /usr/src/nginx-1.15.8 \
    && mkdir /usr/local/nginx \
    && ./configure --prefix=/usr/local/nginx && make && make install \
    && ln -s /usr/local/nginx/sbin/nginx /usr/local/sbin/ \
    && nginx
#删除安装nginx安装目录
RUN rm -rf /usr/src/nginx-nginx-1.15.8
#对外暴露80端口
EXPOSE 80
#启动nginx
CMD ["nginx", "-g", "daemon off;"]
```

# 使用Dockerfile构建镜像

## 构建流程

1. 上传安装包

首先我们需要把要构建的软件安装包上传到服务器中，我们可以在服务器目录上创建一个专门的文件夹，如：/var/nginx_build,然后把从nginx官网下载的nginx-1.15.8.tar.gz安装包上传到这个目录里。

2. 编写Dockerfile

如何编写nginx的Dockerfile上面已经详细介绍，现在我们只需把编写好的Dockerfile上传到/var/nginx_build目录下，当然你也可以在服务器上直接编写Dockerfile，但是要记得一定保证Dockerfile文件和安装包在一个目录下。

3. 运行构建命令构建

docker build 命令用于使用 Dockerfile 创建镜像。

格式：

```
docker build [OPTIONS] PATH | URL | -
```

OPTIONS有很多指令，下面列举几个常用的：

> - --build-arg=[] :设置镜像创建时的变量；
> - -f :指定要使用的Dockerfile路径；
> - --force-rm :设置镜像过程中删除中间容器；
> - --rm :设置镜像成功后删除中间容器；
> - --tag, -t: 镜像的名字及标签，通常 name:tag 或者 name 格式；