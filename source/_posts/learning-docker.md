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
### 一、核心概念与原理
1. Docker与虚拟机的区别
  - 隔离级别：Docker是进程级隔离，共享宿主机内核；虚拟机是硬件级隔离，需运行完整操作系统
  - 资源消耗：Docker启动秒级、资源占用低（MB级）；虚拟机启动分钟级、资源消耗高（GB级）
  - 性能：Docker接近原生性能，虚拟机存在额外虚拟化开销
2. Docker安全性
  - 依赖Linux内核安全特性（命名空间、cgroups、AppArmor/SELinux）实现隔离
  - 镜像签名机制保障镜像来源可信
  - 生产环境验证表明隔离性弱于虚拟机但安全性仍较高

### 二、容器与镜像管理
1. 容器生命周期
  状态：运行(Running)、暂停(Paused)、重启(Restarting)、退出(Exited)
  常用命令：
  ```shell
  docker start/stop/restart <容器ID>  # 启停容器 
  docker rm $(docker ps -aq)        # 清理停止的容器
  docker logs <容器ID>              # 查看日志
  ```

2. **数据持久化**  
  - 使用卷(Volume)实现数据持久化：`docker run -v /宿主机路径:/容器路径`  
  - 容器退出后数据保留，需手动删除容器才会清除  

### 镜像管理
1. **镜像构建原则**  
  - 精简层级，避免冗余依赖  
  - 使用.dockerignore过滤无关文件  
  - 指定明确版本号（如`FROM ubuntu:20.04`）  
2. **Dockerfile指令**  
  - 高频指令：`FROM`、`RUN`、`COPY`、`ADD`、`EXPOSE`、`CMD`  
  - **COPY vs ADD**：  
    - COPY直接复制文件  
    - ADD支持自动解压压缩包和远程URL下载  

---

## 三、高级特性与工具
1. **Docker Compose**  
  - 用途：通过YAML文件定义多容器应用  
  - 启动命令：`docker-compose up -d`  
2. **Docker Swarm**  
  - 集群管理工具，支持服务弹性伸缩  
  - 节点类型：Manager（管理集群）、Worker（运行容器）  
3. **网络模式**  
  - **bridge**：默认桥接网络  
  - **host**：共享宿主机网络栈  
  - **overlay**：跨主机容器通信  

---

## 四、生产环境实践
1. **资源控制**  
  - 通过`--cpus`限制CPU，`--memory`限制内存  
  - 使用cgroups实现资源隔离  
2. **监控方案**  
  - 基础命令：`docker stats`实时监控资源使用  
  - 日志收集：`docker logs`结合ELK等工具  
3. **环境迁移**  
  - 步骤：停止Docker服务→复制/var/lib/docker目录→调整新宿主机配置  

---

## 五、进阶问题
1. **容器化应用类型**  
  - 优先无状态应用（如Web服务），有状态应用需配合持久化存储  
2. **非Linux系统运行原理**  
  - Mac/Windows通过Linux虚拟机运行容器（如Hyper-V、VirtualBox）  
3. **CI/CD集成**  
  - 镜像作为标准化交付物，实现开发/测试/生产环境一致性  


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
通俗的讲，**镜像是一个只读的文件和文件夹组合，是Docker容器启动的先决条件。**
镜像是一个只读的文件和文件夹组合，包含了容器运行时所需要的所有基础文件和配置信息，是容器启动的基础。
镜像不包括任何动态数据。

通常情况下，可以通过下面两种方式创建镜像：
- 基于centos镜像制作自己的业务镜像
  - 安装nginx服务
  - 部署应用程序
  - 做一些自定义配置
- 从功能镜像仓库拉取别人制作好的镜像
  - 常用的软件或者系统，例如nginx、ubuntu、centos、mysql等，可以到Docker Hub搜索并下载。

##### 镜像构建
直接看图，有下面几种：
![镜像操作](4f507556/镜像操作.jpg)

```shell
# Docker镜像的拉取使用docker pull命令
docker pull [Registry]/[Repository]/[Image]:[Tag]
# 使用docker tag命令将镜像重命名，这种只是建了一个别名，底层指向的还是同一个imageid
docker tag busybox:latest mybusybox:latest
# 使用docker rmi 或者 docker image rm 命令删除镜像
```

构建方法：
- 使用docker commit命令从运行中的容器提交为镜像
- 使用docker build命令从Dockerfile构建镜像：主要的构建方式

##### Dockerfile文件
- Dockerfile的每一行命令都会生成一个对立的镜像层，并且拥有唯一的ID
- Dockerfile的命令是完全透明的，通过查看Dockerfile的内容，可以知道镜像是如何创建的
- Dockerfile是纯文本的，方便跟随代码一起存放在代码仓库并管理

**相关指令简介：**
| Dockerfile指令        | 指令说明           |
| ------------- |:-------------:|
|  FROM      | Dockerfile除了注释第一行必须是FROM，FROM后面跟镜像名称，代码我们要基于哪个基础镜像构建我们的容器 |
| RUN      | RUN后面跟一个具体的命令，类似于Linux命令行执行命令      |
| ADD      | 拷贝本机文件或者远程文件到镜像内      |
| COPY      | 拷贝本机文件的到镜像内      |
| USER      | 指定容器启动用户      |
| ENTRYPOINT      | 容器的启动命令      |
| CMD      | CMD为ENTRYPOINT指令提供默认参数，也可以单独使用CMD指定容器启动参数      |
| ENV      | 指定容器运行时的环境变量，格式为key=value      |
| ARG      | 定义外部变量，构建镜像时可以使用build-arg <varname>=<value>的格式传递参数用于构建      |
| EXPOSE      | 指定容器监听的端口，格式为[port]/tcp或者[port]/udp      |
| WORKDIR      | 为Dockerfile中跟在其后的所有RUN、CMD、ENTRYPOINT、COPY和ADD命令设置工作目录      |

示例：
```shell
FROM centos:7
COPY nginx.repo /etc/yum.repos.d/nginx.repo
RUN yum install -y nginx
EXPOSE 80
ENV HOST=mynginx
CMD ["nginx", "-g", "daemon off"]
```

##### 实现原理
Docker镜像是由一系列镜像层（layer）组成的，每一层代表了镜像构建过程中的一次提交。
一个示例：
![一个镜像分层的示例](4f507556/镜像分层的示例.jpg)

#### 容器
通俗的讲，容器是镜像的运行实体，而镜像是静态的只读文件，容器带有运行时，即容器运行着真正的应用进程。
容器有初建、运行、停止、暂停和删除五种状态
在容器内部，无法看到主机上的进程、环境变量、网络等信息
![容器组成示意图](4f507556/容器组成示意图.jpg)

##### 容器的生命周期
1. created: 初建状态
2. running: 运行状态
3. stopped: 停止状态
4. paused: 暂停状态
5. deleted: 删除状态

![容器的生命周期](4f507556/容器的生命周期.jpg)


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