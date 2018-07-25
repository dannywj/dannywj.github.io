---
title: 神奇的Docker
tags: develop
date: 2018-07-24 16:31:57
---
# Docker简介
## Docker定义与理解
Docker 属于 Linux 容器的一种封装，提供简单易用的容器使用接口。
也可以理解为一种轻量级的虚拟机,可以方便的生成或组合出符合我们需要的环境.
## 用途
Docker 的主要用途，目前有三大类。

（1）提供一次性的环境。比如，本地测试他人的软件、持续集成的时候提供单元测试和构建的环境。

（2）提供弹性的云服务。因为 Docker 容器可以随开随关，很适合动态扩容和缩容。

（3）组建微服务架构。通过多个容器，一台机器可以跑多个服务，因此在本机就可以模拟出微服务架构。
# Docker安装
以Ubuntu 17.10版本环境为例进行说明
## 卸载旧版本
旧版本的 Docker 称为 docker 或者 docker-engine，使用以下命令卸载旧版本：
```
$ sudo apt-get remove docker \
               docker-engine \
               docker.io
```
## 使用 APT 安装
由于 apt 源使用 HTTPS 以确保软件下载过程中不被篡改。因此，我们首先需要添加使用 HTTPS 传输的软件包以及 CA 证书。
```
$ sudo apt-get update

$ sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common
```

鉴于国内网络问题，强烈建议使用国内源，官方源请在注释中查看。

为了确认所下载软件包的合法性，需要添加软件源的 GPG 密钥。
```
$ curl -fsSL https://mirrors.ustc.edu.cn/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
```
然后，我们需要向 source.list 中添加 Docker 软件源
```
$ sudo add-apt-repository \
    "deb [arch=amd64] https://mirrors.ustc.edu.cn/docker-ce/linux/ubuntu \
    $(lsb_release -cs) \
    stable"
```
`警告：切勿在没有配置 Docker APT 源的情况下直接使用 apt 命令安装 Docker.`

安装 Docker CE
更新 apt 软件包缓存，并安装 docker-ce：
```
$ sudo apt-get update

$ sudo apt-get install docker-ce
```
安装完成后，运行下面的命令，验证是否安装成功。
```
$ docker version
# 或者
$ docker info
```
# 启动 Docker CE
```
$ sudo systemctl enable docker
$ sudo systemctl start docker
```

Docker 需要用户具有 sudo 权限，为了避免每次命令都输入sudo，可以把用户加入 Docker 用户组（官方文档）。
```
$ sudo usermod -aG docker $USER
```

hello world
```
$ docker run hello-world
```
第一次运行会从仓库下载该映像，下载成功后会输出：
```
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
 https://docs.docker.com/engine/userguide/

```

# 常用命令
```
$ docker image ls
```

image 文件生成的容器实例，本身也是一个文件，称为容器文件。也就是说，一旦容器生成，就会同时存在两个文件： image 文件和容器文件。而且关闭容器并不会删除容器文件，只是容器停止运行而已。
```
# 列出本机正在运行的容器
$ docker container ls

# 列出本机所有容器，包括终止运行的容器
$ docker container ls --all
```

```
danny@danny-parallels:~$ docker container ls --all
CONTAINER ID        IMAGE               COMMAND                  CREATED              STATUS                          PORTS               NAMES
2ac511f9e2d7        hello-world         "/hello"                 55 seconds ago       Exited (0) 54 seconds ago                           unruffled_bhabha
77245f2dd6b7        hello-world         "/hello"                 About a minute ago   Exited (0) About a minute ago                       friendly_pare
7d952cca3ba6        hello-world         "/hello"                 8 minutes ago        Exited (0) 8 minutes ago                            awesome_stallman
c26800116685        ubuntu:14.04        "/bin/bash"              4 days ago           Exited (0) 4 days ago                               modest_allen
fe278ae1ae84        ubuntu:14.04        "/bin/bash"              4 days ago           Exited (0) 4 days ago                               dreamy_swartz
24ee5515501c        wordpress           "docker-entrypoint.s…"   4 days ago           Exited (0) 4 days ago                               docker-demo_web_1
6e164b64e3ac        mysql:5.7           "docker-entrypoint.s…"   4 days ago           Exited (0) 4 days ago                               docker-demo_mysql_1
4d41741e67c1        hello-world         "/hello"                 4 days ago           Exited (0) 4 days ago                               adoring_kilby
814df7693f7e        hello-world         "/hello"                 4 days ago           Exited (0) 4 days ago                               laughing_pasteur
29f12178f175        hello-world         "/hello"                 4 days ago           Exited (0) 4 days ago                               sharp_shannon
15cb57f17715        hello-world         "/hello"                 4 days ago           Exited (0) 4 days ago                               focused_gates
044e9fd2ffeb        hello-world         "/hello"                 4 days ago           Exited (0) 4 days ago                               vibrant_hugle

```
上面命令的输出结果之中，包括容器的 ID。很多地方都需要提供这个 ID，比如上一节终止容器运行的`docker container kill`命令。

终止运行的容器文件，依然会占据硬盘空间，可以使用`docker container rm`命令删除。

```
$ docker container rm [containerID]
```
# Docker网站搭建
## 新建 MySQL 容器
```
$ docker container run \
  -d \
  --rm \
  --name wordpressdb \
  --env MYSQL_ROOT_PASSWORD=123456 \
  --env MYSQL_DATABASE=wordpress \
  mysql:5.7
```
上面的命令会基于 MySQL 的 image 文件（5.7版本）新建一个容器。
**参数含义:**
-d：容器启动后，在后台运行。
--rm：容器终止运行后，自动删除容器文件。
--name wordpressdb：容器的名字叫做wordpressdb
--env MYSQL_ROOT_PASSWORD=123456：向容器进程传入一个环境变量MYSQL_ROOT_PASSWORD，该变量会被用作 MySQL 的根密码。
--env MYSQL_DATABASE=wordpress：向容器进程传入一个环境变量MYSQL_DATABASE，容器里面的 MySQL 会根据该变量创建一个同名数据库（本例是WordPress）。

查看正在运行的容器
```
$ docker container ls
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
7bc3aa636365        phpwithmysql        "docker-php-entrypoi…"   4 minutes ago       Up 4 minutes        80/tcp              wordpress
a1972f622329        mysql:5.7           "docker-entrypoint.s…"   4 minutes ago       Up 4 minutes        3306/tcp            wordpressdb

$ docker container logs wordpressdb
```
## 定制 PHP 容器
官方php容器不带mysql,因此需要自己安装一下mysql扩展,必须自己新建 image 文件。

新建一个Dockerfile文件，写入下面的内容
```
FROM php:5.6-apache
RUN docker-php-ext-install mysqli
CMD apache2-foreground
```
上面代码的意思，就是在原来 PHP 的 image 基础上，安装mysqli的扩展。然后，启动 Apache。

基于这个 Dockerfile 文件，新建一个名为phpwithmysql的 image 文件。

```
$ docker build -t phpwithmysql .
```

## 容器链接mysql
现在基于 phpwithmysql image，重新新建一个 WordPress 容器。

```
$ docker container run \
  --rm \
  --name wordpress \
  --volume "$PWD/":/var/www/html \
  --link wordpressdb:mysql \
  phpwithmysql
```
跟上一次相比，上面的命令多了一个参数`--link wordpressdb:mysql`，表示 WordPress 容器要连到wordpressdb容器，冒号表示该容器的别名是mysql。

这时还要改一下`wordpress目录的权限`，让容器可以将配置信息写入这个目录（容器内部写入的/var/www/html目录，会映射到这个目录）。

# Docker Compose 体验
各个容器的关联启动每次都要手工进行操作,略显繁琐.Docker 提供了一种更简单的方法，来管理多个容器的联动。
需要定义一个 YAML 格式的配置文件`docker-compose.yml`，写好多个容器之间的调用关系。然后，只要一个命令，就能同时启动/关闭这些容器。
```
# 启动所有服务
$ docker-compose up
# 关闭所有服务
$ docker-compose stop
```
## docker-compose安装
在 Linux 上的也安装十分简单，从 官方 GitHub Release 处直接下载编译好的二进制文件即 可。 例如，在 Linux 64 位系统上直接下载对应的二进制包。
```
$ sudo curl -L https://github.com/docker/compose/releases/download/1.17.1/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose

$ sudo chmod +x /usr/local/bin/docker-compose
```
新建`docker-compose.yml`文件示例
```
mysql:
    image: mysql:5.7
    environment:
     - MYSQL_ROOT_PASSWORD=123456
     - MYSQL_DATABASE=wordpress
web:
    image: wordpress
    links:
     - mysql
    environment:
     - WORDPRESS_DB_PASSWORD=123456
    ports:
     - "127.0.0.3:8080:80"
    working_dir: /var/www/html
    volumes:
     - wordpress:/var/www/html
```
两个顶层标签表示有两个容器mysql和web
```
$ docker-compose up
```
浏览器访问 http://127.0.0.3:8080，应该就能看到 WordPress 的安装界面。

现在关闭两个容器。
```
$ docker-compose stop
```
关闭以后，这两个容器文件还是存在的，写在里面的数据不会丢失。下次启动的时候，还可以复用。下面的命令可以把这两个容器文件删除（容器必须已经停止运行）。

```
$ docker-compose rm
```

启动示例:
```
danny@danny-parallels:~/docker-demo$ docker-compose up
Starting docker-demo_mysql_1 ... done
Starting docker-demo_web_1   ... done
Attaching to docker-demo_mysql_1, docker-demo_web_1
mysql_1  | 2018-07-25T02:22:28.873206Z 0 [Warning] TIMESTAMP with implicit DEFAULT value is deprecated. Please use --explicit_defaults_for_timestamp server option (see documentation for more details).
......
```
参考资料:
Docker 入门教程
http://www.ruanyifeng.com/blog/2018/02/docker-tutorial.html

Docker 微服务教程(php+mysql+WordPress)
http://www.ruanyifeng.com/blog/2018/02/docker-wordpress-tutorial.html

Docker 指导手册
https://docs.docker.com/compose/install/#install-compose
