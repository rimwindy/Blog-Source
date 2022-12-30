---
title: Docker 安装 MySQL 手记
tags: [Docker, MySQL]
date: 2019-12-11 15:15
categories: 笔记本
# thumbnail: /img/2019/12/11/0.jpg
---
Docker 是一个开源的应用容器引擎，基于 Go 语言编写，是目前最流行的容器解决方案。使用Docker可以将应用及依赖打包到一个可移植的容器中，然后发布到任何流行的 Windows 或 Linux 机器上运行。由于其完全使用沙盒机制，真正实现了应用程序与基础架构的分离，且与传统的虚拟机相比，Docker 的性能开销也极低。

> *Debug your app, not your environment.*

# Docker简介
Docker 主要有三个重要的基本概念，分别是镜像（Image）、容器（Container）和仓库（Repository）。

* 镜像
Docker 镜像（Image）是一个特殊的文件系统，除了提供容器运行时所需的程序、库、资源、配置等文件外，还包含了一些为运行时准备的一些配置参数（如匿名卷、环境变量、用户等）。镜像不包含任何动态数据，其内容在构建之后也不会被改变。

* 容器
镜像（Image）和容器（Container）的关系，就像是面向对象程序设计中的类和对象一样，镜像是静态的定义，容器是镜像运行时的实体。容器可以被创建、启动、停止、删除、暂停等。

* 仓库
镜像构建完成后，可以很容易的在当前宿主机上运行，但是，如果需要在其它服务器上使用这个镜像，我们就需要一个集中的存储、分发镜像的服务，通常可以选择[Docker Hub](https://hub.docker.com)，如果有私有仓库的需要的话，可以选择使用官方提供的Docker Registry服务。

# 安装Docker
因为最近学校开设了 JSP 的实验课，需要用到数据库服务，但是安装一个完整的数据库需要很大的硬盘空间（SQL Server 需要10G，下载还贼慢，真是醉了...），而且很多功能也用不上，所以我就选择了使用轻量化的 Docker 来运行 MySQL，下面记录一下具体操作：

## 开启虚拟化
> 如果是Windows的话，需要64位机器、Windows 10的系统，然后需要至少4G的内存，具体的安装要求可以参考[官网](https://docs.docker.com/docker-for-windows/install)的介绍。

首先需要查看计算机有没有开启虚拟化（如果你之前安装过手游模拟器的话，大概率会提示你开启VT的，这里的虚拟化就是指这个东西~），查看方法也非常简单，打开任务管理器的性能标签页即可查看。如果没有开启的话需要进BIOS开启一下，否则Docker无法使用。
![VT](/img/2019/12/11/1.png)

另外还需要 Hyper-V 的支持，不过 Docker 会自动帮我们开启，所以也不需要手动开启了。

## 安装
去[官网](https://www.docker.com)下载安装包即可，安装完成后会提示重启，方便系统添加Docker需要的组件。

## 配置镜像加速
重启完成后你应该就能在系统托盘上看到Docker的图标了，不过因为众所周知的原因，在国内拉取Docker Hub上的镜像速度不是很理想，因此可以选择国内的加速服务，比如我这里使用了网易的：http://hub-mirror.c.163.com

选择Settings-Daemon-Registry mirrors，将镜像站点的链接粘贴进去即可：
![Registry mirrors](/img/2019/12/11/2.png)

配置完成后需要重新启动Docker生效。

# 下载并运行MySQL镜像
可以去Docker Hub搜索相应的镜像，也可以在terminal中使用docker search来搜索，比如：
```bash
docker search mysql
```
![docker search](/img/2019/12/11/3.png)

可以看到返回了镜像名称、镜像介绍、star数等信息，我们可以选择官方的镜像进行下载，使用如下命令：
```bash
# 可以自定义拉取镜像的版本，默认为最新版本
docker pull mysql
```
安装完成后可以使用docker images来查看已有的镜像：
![images](/img/2019/12/11/4.png)

使用如下命令启动MySQL镜像：
```bash
docker run --name MySQL_Demo -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 -d mysql
```
返回一串 CONTAINER ID，代表已经启动成功。接下来就可以使用IDE或Navicat等工具连接你的数据库了。
![docker run](/img/2019/12/11/5.png)

# 常用指令

## 查看已运行的容器
```bash
docker ps -a
```

## 进入mysql容器
```bash
docker exec -it MySQL_Demo bash
```

## 启动、停止、杀死容器
```bash
docker start|stop|kill Name/ID
```

## 删除镜像
```bash
docker rmi Name/ID
```

## 删除容器
```bash
docker rm Name/ID
```

*E.N.D*