---
title: docker学习实战
abbrlink: 4f507556
date: 2024-12-12 16:04:00
updated: 2024-12-12 16:04:00
tags:
  - docker
  - linux
categories: 云原生
---

## 前言
梳理一些关于docker相关的知识
<!-- more -->

## 基础概念相关
### Docker能做什么

### Docker安装
centos7 安装
```shell
# 如果安装旧版本，可以先卸载
sudo yum remove docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine
# 首次安装添加docker安装源
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
# 安装最新版本的docker
sudo yum install docker-ce docker-ce-cli containerd.io
## 查看安装的版本
sudo yum list docker-ce --showduplicates | sort -r
## 指定安装版本
sudo yum install docker-ce-<version_string> docker-ce-cli-<version_string> containerd.io
# 启动docker
sudo systemctl start docker
# 启动一个hello world容器
sudo docker run hello-world
```
运行结果
```shell
[kemi@localhost ~]$ sudo docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
c1ec31eb5944: Pull complete 
Digest: sha256:5b3cc85e16e3058003c13b7821318369dad01dac3dbb877aac3c28182255c724
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.
```

### 关于chroot
chroot是在Unix和Linux系统的一个操作，针对正在运行的软件进程和它的子进程，改变它外显的根目录。
一个运行在这个环境下，经由chroot设置根目录的程序，它不能够对这个指定根目录之外的文件进行访问动作，不能读取，也不能更改它的内容。
一个示例：
```shell
# 在rootfs下创建了一些目录和放置了一些二进制文件
cd rootfs
docker export $(docker create busybox) -o busybox.tar
tar -xf busybox.tar
# 执行完之后的结果
[root@localhost rootfs]# tar -xf busybox.tar 
[root@localhost rootfs]# ls -ltr
total 4416
drwxr-xr-x. 4 root  root       30 Sep 26 17:31 var
drwxr-xr-x. 4 root  root       29 Sep 26 17:31 usr
drwxrwxrwt. 2 root  root        6 Sep 26 17:31 tmp
drwx------. 2 root  root        6 Sep 26 17:31 root
lrwxrwxrwx. 1 root  root        3 Sep 26 17:31 lib64 -> lib
drwxr-xr-x. 2 root  root      213 Sep 26 17:31 lib
drwxr-xr-x. 2 65534 65534       6 Sep 26 17:31 home
drwxr-xr-x. 2 root  root    12288 Sep 26 17:31 bin
drwxr-xr-x. 2 root  root        6 Dec 12 06:48 sys
drwxr-xr-x. 2 root  root        6 Dec 12 06:48 proc
drwxr-xr-x. 3 root  root      160 Dec 12 06:48 etc
drwxr-xr-x. 4 root  root       43 Dec 12 06:48 dev
-rw-------. 1 root  root  4505600 Dec 12 06:48 busybox.tar
```
此时通过一个chroot命令，启动一个sh进程，并且把这个rootfs作为sh进程的根目录
```shell
$ chroot /home/kemi/rootfs /bin/sh
```
此时还不能称之为一个完整的容器，因为网路等信息还没有隔离：需要通过linux的**namespace、Cgroup、联合文件系统**来具体实现，具体的职责大致是：
- namespace：主机名、网络、pid等资源的隔离
- Cgroup: 对进程或者进程组作资源的限制
- 联合文件系统：用于镜像构建和容器运行环境

### 关于namespace
Namespace对内核资源进行隔离，使得容器中的进程都可以在单独的命名空间中运行，并且只可以访问当前容器命名空间的资源。
Namespace可以隔离进程ID、主机名、用户ID、文件名、网络访问和进程间通信等相关资源
其中docker用到下面5中namespace：
- **pid namespace**: 隔离进程ID
- **net namespace**: 隔离网络接口
- **mnt namespace**: 文件系统挂载点隔离
- **ipc namespace**: 信号量，消息队列和共享内存的隔离
- **uts namespace**: 主机名和域名的隔离

### 关于cgroup
cgroup是一种linux内核功能，可以限制和隔离进程的资源使用情况（如CPU、内存、磁盘I/O、网络等）
### 关于联合文件系统
又叫UnionFS，是一种通过创建文件层进程操作的文件系统
docker利用改技术，为容器提供构建层，使得容器可以实现写时复制以及镜像的分层和存储
常用的联合文件系统有AUFS、Overlay和Devicemapper等

## 实现原理及关键技术
### 核心概念
docker核心围绕：镜像、容器、仓库来构建的
#### 镜像
通俗的讲，镜像是一个只读的文件和文件夹组合，是Docker容器启动的先决条件。
镜像是一个只读的文件和文件夹组合，包含了容器运行时所需要的所有基础文件和配置信息，是容器启动的基础。

通常情况下，可以通过下面两种方式创建镜像：
- 基于centos镜像制作自己的业务镜像
  - 安装nginx服务
  - 部署应用程序
  - 做一些自定义配置
- 从功能镜像仓库拉取别人制作好的镜像
  - 常用的软件或者系统，例如nginx、ubuntu、centos、mysql等，可以到Docker Hub搜索并下载。

#### 容器
通俗的讲，容器是镜像的运行实体，而镜像是静态的只读文件，容器带有运行时，即容器运行着真正的应用进程。
容器有初建、运行、停止、暂停和删除五种状态
在容器内部，无法看到主机上的进程、环境变量、网络等信息

#### 仓库
类似于代码仓库，Docker的镜像仓库用来存储和分发Docker镜像，其中可以分为公共镜像仓库和私有镜像仓库。

### Docker架构
![Docker架构图](4f507556/docker架构图.jpg)
经典的C/S架构模式，客户端负责发送指令，服务的负责接收和处理指令。

Docker客户端是一种泛称：
- Docker命令是Docker用户与Docker服务端交互的主要方式
- 使用直接请求Rest API方式与Docker服务端交互
- 使用各种语言的SDK与Docker服务端交互

Docker服务端是Docker所有后台服务的统称：其中dockerd负责响应和处理来自Docker客户端的请求，然后将客户端的请求转化为Docker的具体操作。

## 容器编排相关

## 综合应用