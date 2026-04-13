---
title: docker学习实战
abbrlink: 4f507556
date: 2024-12-12 16:04:00
updated: 2026-01-28 15:50:58
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


---

## Docker安装(centos7)

### 1. 卸载旧版本
首先如果系统中已经存在旧的Docker，则先卸载
```shell
yum remove docker \
    docker-client \
    docker-client-latest \
    docker-common \
    docker-latest \
    docker-latest-logrotate \
    docker-logrotate \
    docker-engine \
    docker-selinux
```

### 2. 配置Docker的yum库
```shell
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
```

如果安装失败，使用阿里云更新yum源（通常应该不会，安装好centos第一件事，通常得换源的 :) ）
- 换源：
  ```shell
  curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
  ```
- 清除yum:
  ```shell
  yum clean all
  ```
- 更新缓存
  ```shell
  yum makecache
  ```

### 3. 配置Docker的yum源
```shell
sudo yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
 
sudo sed -i 's+download.docker.com+mirrors.aliyun.com/docker-ce+' /etc/yum.repos.d/docker-ce.repo
```

### 4. 更新yum，建立缓存
```shell
sudo yum makecache
```

### 5. 安装Docker
```shell
yum install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 6. 启动和校验
```shell
# 启动Docker
systemctl start docker
 
# 停止Docker
systemctl stop docker
 
# 重启
systemctl restart docker
 
# 设置开机自启
systemctl enable docker
 
# 执行docker ps命令，如果不报错，说明安装启动成功
# 正常应该看到下面一行：
# CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
docker ps
```

### 7. 配置镜像加速
```shell

# 创建目录
mkdir -p /etc/docker
 
# 复制内容(用的tee命令，你也可以用常用的vim，copy下面的json串写到文件里面即可)
tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": [
    "https://docker.m.daocloud.io/",
    "https://hub-mirror.c.163.com",
    "https://dockerproxy.com/",
    "https://mirror.baidubce.com/",
    "https://docker.nju.edu.cn/",
    "https://docker.mirrors.sjtug.sjtu.edu.cn/",
    "https://mirror.ccs.tencentyun.com",
    "https://docker-0.unsee.tech",
    "https://register.liberx.info/",
    "https://docker.registry.cyou/",
    "https://docker-cf.registry.cyou/",
    "https://dockercf.jsdelivr.fyi/",
    "https://docker.jsdelivr.fyi/",
    "https://dockertest.jsdelivr.fyi/",
    "https://mirror.iscas.ac.cn/",
    "https://docker.rainbond.cc/",
    "https://mirror.aliyuncs.com",
    "https://docker.mirrors.ustc.edu.cn/"
  ]
}
EOF
 
# 重新加载配置
systemctl daemon-reload
 
# 重启Docker
systemctl restart docker
```

## 在ubuntu24中安装
PS：卸载相关，参考：[知乎文章](https://zhuanlan.zhihu.com/p/1919709942706837199)，其中有关于卸载相关的介绍。
### 更新包索引并安装必要的依赖包
```shell
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg
```

### 添加 Docker 官方 GPG 密钥
```shell
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

### 添加 Docker 官方仓库
```shell
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### 再次更新包索引并安装 Docker 引擎
```shell
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 配置非 root 用户访问（可选但推荐）
```shell
sudo usermod -aG docker $USER
newgrp docker
```

### 配置国内 Docker 镜像源
```shell
# 创建或编辑Docker配置文件
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": [
    "https://docker.m.daocloud.io/",
    "https://hub-mirror.c.163.com",
    "https://dockerproxy.com/",
    "https://mirror.baidubce.com/",
    "https://docker.nju.edu.cn/",
    "https://docker.mirrors.sjtug.sjtu.edu.cn/",
    "https://mirror.ccs.tencentyun.com",
    "https://docker-0.unsee.tech",
    "https://register.liberx.info/",
    "https://docker.registry.cyou/",
    "https://docker-cf.registry.cyou/",
    "https://dockercf.jsdelivr.fyi/",
    "https://docker.jsdelivr.fyi/",
    "https://dockertest.jsdelivr.fyi/",
    "https://mirror.iscas.ac.cn/",
    "https://docker.rainbond.cc/",
    "https://mirror.aliyuncs.com",
    "https://docker.mirrors.ustc.edu.cn/"
  ]
}
EOF

# 重新加载Docker配置并重启服务
sudo systemctl daemon-reload
sudo systemctl restart docker
```

### 验证镜像配置生效（和6. 一同执行）
```shell
# 检查Docker配置
sudo docker info | grep -A 1 'Registry Mirrors'
# 检查 Docker 服务状态
sudo systemctl status docker
# 验证一下
sudo docker run hello-world
# 如果没有hello-world，docker pull 拉一下
```

![docker_status](4f507556/docker_status.jpg)

## Docker基础
docker官方网站：https://docs.docker.com/
### 常用命令
| 命令        | 说明           |
| ------------- |:-------------:|
|  docker  pull      | 拉取镜像 |
| docker push      | 推送镜像到DockerRegistry      |
| docker images      | 查看本地镜像      |
| docker rmi      | 删除本地镜像      |
| docker run      | 创建并运行容器(不能重复创建)      |
| docker stop      | 停止指定容器      |
| docker start      | 启动指定容器      |
| docker restart      | 重新启动容器      |
| docker rm      | 删除指定容器      |
| docker ps      | 查看容器      |
| docker logs      | 查看容器运行日志      |
| docker exec      | 进入容器      |
| docker save      | 保存镜像到本地压缩文件      |
| docker load      | 加载本地压缩文件到镜像      |
| docker inspect      | 查看容器详细信息      |

总结开来，可以用下图所示：
![Docker命令执行](4f507556/Docker命令执行.jpg)

默认情况下，每次重启虚拟机我们都需要手动启动Docker和Docker中的容器。通过命令可以实现开机自启：
```shell
# Docker开机自启
systemctl enable docker
 
# Docker容器开机自启
docker update --restart=always [容器名/容器id]
```

### 一则常用演示(nginx)
```shell
# 第1步，去DockerHub查看nginx镜像仓库及相关信息
 
# 第2步，拉取Nginx镜像
docker pull nginx
 
# 第3步，查看镜像
docker images
# 结果如下：
REPOSITORY   TAG       IMAGE ID       CREATED       SIZE
nginx        latest    3f8a4339aadd   7 years ago   108MB
 
# 第4步，创建并允许Nginx容器
docker run -d --name nginx -p 80:80 nginx
 
# 第5步，查看运行中容器
docker ps
# 也可以加格式化方式访问，格式会更加清爽
docker ps --format "table {{.ID}}\t{{.Image}}\t{{.Ports}}\t{{.Status}}\t{{.Names}}"
 
# 第6步，访问网页，地址：http://虚拟机地址
 
# 第7步，停止容器
docker stop nginx
 
# 第8步，查看所有容器
docker ps -a --format "table {{.ID}}\t{{.Image}}\t{{.Ports}}\t{{.Status}}\t{{.Names}}"
 
# 第9步，再次启动nginx容器
docker start nginx
 
# 第10步，再次查看容器
docker ps --format "table {{.ID}}\t{{.Image}}\t{{.Ports}}\t{{.Status}}\t{{.Names}}"
 
# 第11步，查看容器详细信息
docker inspect nginx
 
# 第12步，进入容器,查看容器内目录
docker exec -it nginx bash
# 或者，可以进入MySQL
docker exec -it mysql mysql -uroot -p
 
# 第13步，删除容器
docker rm nginx
# 发现无法删除，因为容器运行中，强制删除容器
docker rm -f nginx
```

## Docker其它知识

### ~~Docker安装~~
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
|:-------------:|:-------------:|
|  `FROM`      | Dockerfile除了注释第一行必须是FROM，FROM后面跟镜像名称，代码我们要基于哪个基础镜像构建我们的容器 |
| `RUN`      | RUN后面跟一个具体的命令，类似于Linux命令行执行命令      |
| `ADD`      | 拷贝本机文件或者远程文件到镜像内      |
| `COPY`      | 拷贝本机文件的到镜像内      |
| `USER`      | 指定容器启动用户      |
| `ENTRYPOINT`      | 容器的启动命令      |
| `CMD`      | CMD为ENTRYPOINT指令提供默认参数，也可以单独使用CMD指定容器启动参数      |
| `ENV`      | 指定容器运行时的环境变量，格式为key=value      |
| `ARG`      | 定义外部变量，构建镜像时可以使用build-arg <varname>=<value>的格式传递参数用于构建      |
| `EXPOSE`      | 指定容器监听的端口，格式为[port]/tcp或者[port]/udp      |
| `WORKDIR`      | 为Dockerfile中跟在其后的所有RUN、CMD、ENTRYPOINT、COPY和ADD命令设置工作目录      |

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

## 综合应用(环境搭建)
### 搭建Postgresql
#### 1. 拉取镜像
```shell
docker pull postgres

## 拉取后
[root@localhost ~]# docker images
REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
postgres     latest    f49abb9855df   2 months ago   438MB
nginx        latest    3f8a4339aadd   7 years ago    108MB
```

#### 2. 创建本地卷
Docker 会自动在 /var/lib/docker/volume/ 路径下为主机上的卷创建一个目录。该卷可以在容器之间共享和重用， 且默认会一直存在。
```shell
docker volume list            # 列出 Docker 卷
docker volume rm pgdata       # 删除 Docker 卷

docker volume create pgdata   # 创建 Docker 卷
docker volume inspect pgdata  # 检查 Docker 卷

[root@localhost ~]# docker volume inspect pgdata
[
    {
        "CreatedAt": "2025-04-28T09:58:07-04:00",
        "Driver": "local",
        "Labels": null,
        "Mountpoint": "/var/lib/docker/volumes/pgdata/_data",
        "Name": "pgdata",
        "Options": null,
        "Scope": "local"
    }
]
```

#### 3. 构建镜像容器
```shell
docker run -it \
  --name postgres \
  --restart always \
  -e TZ='Asia/Shanghai' \
  -e POSTGRES_PASSWORD='your_password' \
  -e ALLOW_IP_RANGE=0.0.0.0/0 \
  -v /home/postgres/data:/var/lib/postgresql \
  -p 55435:5432 \
  -d postgres

```

其中，上述命令分别的含义是：

| 名称 | 解释 |
| :-------------: | :-------------: |
|  `--name`      |自定义容器名称 |
| `--restart always`      | 设置容器在 docker 重启时自动启动容器     |
| `-e POSTGRES_PASSWORD`      | Postgresql 数据库密码      |
| `-e ALLOW_IP_RANGE=0.0.0.0/0`      | 表示允许所有 IP 访问     |
| `-e TZ='Asia/Shanghai'`      | 设置时区     |
| `-v [path] : [path]`      | 本地目录映射 (本地目录 : 容器内路径)      |
| `-p 55435:5432 `     | 端口映射 (主机端口 : 容器端口)      |
| `d postgres `     |   镜像名称      |

#### 4. 进入容器
```shell
docker exec -it postgres bash
````

#### 5. 切换当前用户，再登录数据库
将当前 root 切换成 postgres
```shell
su postgres
```

输入用户名/密码执行完后，再根据提示输入
```shell
psql -U postgres -W
```

输入密码，登录成功
![登录成功](4f507556/MyCapture_2025-04-28_22-12-55.jpg)

#### 6. 创建新用户
根据第五步，先切换到 Linux 用户 postgres，并执行如下 psql。
PS：根据你实际的需要，按需修改即可，都是传统的语句。
```shell
create user nimbusk with password 'your_password';            # 创建数据库新用户
CREATE DATABASE nimbuskdb OWNER nimbusk;                # 创建用户数据库
GRANT ALL PRIVILEGES ON DATABASE nimbuskdb TO nimbusk;  # 将 testdb 数据库的所有权限都赋予 test
\q                                                # 使用命令 \q 退出 psql

```

#### 7. 客户端链接验证
这个没啥，找个你常用的链接验证即可
![Navicat链接验证](4f507556/MyCapture_2025-04-28_22-15-26.jpg)


### 搭建redis伪集群
#### 拉取镜像
PS：笔者在默认使用lasted拉去镜像的时候，遇到了诡异问题，拉去下来的redis版本是3.x的，导致后续通过redis-cli创建集群的时候直接失败了，浪费了不少时间。
```shell
# 指定redis版本
docker pull redis:7.0.2
```

#### 创建数据文件夹
```shell
mkdir -p ~/redis-cluster/{7000,7001,7002,7003,7004,7005}
for port in 7000 7001 7002 7003 7004 7005; do
  mkdir -p ~/redis-cluster/${port}/data
done
```

#### 创建配置文件
先生成一个配置文件模板
```shell
cat > redis-cluster.conf.template << 'EOF'
port ${PORT}
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
appendonly yes
daemonize no
protected-mode no
# cluster-announce-ip 127.0.0.1 让docker自己寻找，注释了
cluster-announce-port ${PORT}
cluster-announce-bus-port 1${PORT}
EOF
```

通过模板文件生成redis配置文件：
```shell
for port in 7001 7002 7003 7004 7005 7006; do
  sed "s/\${PORT}/$port/g" redis-cluster.conf.template > redis-$port.conf
done
```

#### 编写docker-compose.yml文件
直接贴，注意你配置文件和改yml文件所在目录即可，否则会出现问题
```yaml
# 新版本docker compose 不用写 version: "3.8"
services:
  redis-7001:
    image: redis:7.0.2 #这里的版本按照上面pull的版本，否则会重新拉取
    container_name: redis-7001
    ports:
      - "7001:7001"
      - "17001:17001"
    volumes:
      - ./redis-7001.conf:/usr/local/etc/redis/redis.conf
      - ./7001/data:/data
    command: ["redis-server", "/usr/local/etc/redis/redis.conf"]
    networks:
      - redis-cluster-net

  redis-7002:
    image: redis:7.0.2
    container_name: redis-7002
    ports:
      - "7002:7002"
      - "17002:17002"
    volumes:
      - ./redis-7002.conf:/usr/local/etc/redis/redis.conf
      - ./7002/data:/data
    command: ["redis-server", "/usr/local/etc/redis/redis.conf"]
    networks:
      - redis-cluster-net

  redis-7003:
    image: redis:7.0.2
    container_name: redis-7003
    ports:
      - "7003:7003"
      - "17003:17003"
    volumes:
      - ./redis-7003.conf:/usr/local/etc/redis/redis.conf
      - ./7003/data:/data
    command: ["redis-server", "/usr/local/etc/redis/redis.conf"]
    networks:
      - redis-cluster-net

  redis-7004:
    image: redis:7.0.2
    container_name: redis-7004
    ports:
      - "7004:7004"
      - "17004:17004"
    volumes:
      - ./redis-7004.conf:/usr/local/etc/redis/redis.conf
      - ./7004/data:/data
    command: ["redis-server", "/usr/local/etc/redis/redis.conf"]
    networks:
      - redis-cluster-net

  redis-7005:
    image: redis:7.0.2
    container_name: redis-7005
    ports:
      - "7005:7005"
      - "17005:17005"
    volumes:
      - ./redis-7005.conf:/usr/local/etc/redis/redis.conf
      - ./7005/data:/data
    command: ["redis-server", "/usr/local/etc/redis/redis.conf"]
    networks:
      - redis-cluster-net

  redis-7006:
    image: redis:7.0.2
    container_name: redis-7006
    ports:
      - "7006:7006"
      - "17006:17006"
    volumes:
      - ./redis-7006.conf:/usr/local/etc/redis/redis.conf
      - ./7006/data:/data
    command: ["redis-server", "/usr/local/etc/redis/redis.conf"]
    networks:
      - redis-cluster-net

networks:
  redis-cluster-net:
    driver: bridge
```

#### 启动
```shell
# 在docker-compose.yml所在目录执行
docker compose up -d
# 查看运行状态，状态都是UP就对了
docker ps
```

#### 创建redis集群
```shell
docker exec -it redis-7001 redis-cli --cluster create \
redis-7001:7001 \
redis-7002:7002 \
redis-7003:7003 \
redis-7004:7004 \
redis-7005:7005 \
redis-7006:7006 \
--cluster-replicas 1
```

遇到输入yes的，直接输入yes后 ，稍等1-2秒即可。
PS：如果遇到等待时间过长，大概率哪里配置有问题，这种情况下，大概率就是docker-compose.yml中网络相关的配置存在问题。

#### 验证集群状态
随便进入一个节点
```shell
# 连接集群
docker exec -it redis-7001 redis-cli -p 7001 -c

# 在 Redis CLI 中查看集群信息
127.0.0.1:7001> CLUSTER INFO
127.0.0.1:7001> CLUSTER NODES

# 测试数据存储
127.0.0.1:7001> set foo bar
127.0.0.1:7001> get foo
```

#### 销毁
```shell
docker-compose down -v
```

### 搭建zookeeper集群
直接docker-compose.yml文件
```yml
services:
  zk1:
    image: zookeeper:3.9.3
    hostname: zk1
    container_name: zk1
    ports:
      - "2181:2181"
      - "8081:8080"
    environment:
      ZOO_MY_ID: 1
      ZOO_SERVERS: server.1=0.0.0.0:2888:3888;2181 server.2=zk2:2888:3888;2181 server.3=zk3:2888:3888;2181
    volumes:
      - ./zk1/data:/data
      - ./zk1/datalog:/datalog
      - /etc/localtime:/etc/localtime
    networks:
      - zk-net
 
  zk2:
    image: zookeeper:3.9.3
    hostname: zk2
    container_name: zk2
    ports:
      - "2182:2181"
      - "8082:8080"
    environment:
      ZOO_MY_ID: 2
      ZOO_SERVERS: server.1=zk1:2888:3888;2181 server.2=0.0.0.0:2888:3888;2181 server.3=zk3:2888:3888;2181
    volumes:
      - ./zk2/data:/data
      - ./zk2/datalog:/datalog
      - /etc/localtime:/etc/localtime
    networks:
      - zk-net
 
  zk3:
    image: zookeeper:3.9.3
    hostname: zk3
    container_name: zk3
    ports:
      - "2183:2181"
      - "8083:8080"
    environment:
      ZOO_MY_ID: 3
      ZOO_SERVERS: server.1=zk1:2888:3888;2181 server.2=zk2:2888:3888;2181 server.3=0.0.0.0:2888:3888;2181
    volumes:
      - ./zk3/data:/data
      - ./zk3/datalog:/datalog
      - /etc/localtime:/etc/localtime
    networks:
      - zk-net
 
networks:
  zk-net:
    driver: bridge
```

其中：
zookeeper 的集群部署，最主要是配置 environment 环境变量，上面配置的 2 个环境变量含义如下：
- `ZOO_MY_ID` 表示当前 zookeeper 实例在集群中的编号，范围为1-255，所以一个 zookeeper 集群最多有 255 个节点
- `ZOO_SERVERS` 表示当前 zookeeper 实例所在集群中的所有节点的编号、主机名（或IP地址）、端口

zookeeper一共需要用到三个端口：
- `2181`：对客户端程序（如我们开发的 SpringBoot 程序）提供服务的连接端口
- `3888`：zookeeper 集群中的节点，选举 leader 使用的通信端口
- `2888`：集群内各节点之间的通讯端口（Leader监听此端口）

volumes挂载相关目录脚本如下：
```shell
mkdir -p ~/zookeeper/{zk1,zk2,zk3}
for port in zk1 zk2 zk3; do
  mkdir -p ~/zookeeper/${port}/data
  mkdir -p ~/zookeeper/${port}/datalog
done
```

#### 验证安装结果
浏览器直接访问如下地址：
http://your_ip:8081/commands/server_stats
![docker_server_stat1](4f507556/docker_server_stat1.jpg)
![docker_server_stat2](4f507556/docker_server_stat2.jpg)
![docker_server_stat3](4f507556/docker_server_stat3.jpg)


### 安装MQTT消息中间件

#### 1. 准备工作结构
在 Ubuntu 上执行以下命令，一次性创建目录和初始密码文件：

```bash
# 创建目录
mkdir -p ~/mqtt/config ~/mqtt/data ~/mqtt/log

# 创建一个空的密码文件并设置权限
touch ~/mqtt/config/pwfile
chmod 600 ~/mqtt/config/pwfile
```


#### 2. 编写集成配置文件
我们需要修改 `mosquitto.conf` 以启用身份验证。

```bash
nano ~/mqtt/config/mosquitto.conf
```

**粘贴以下完整配置：**
```text
# 持久化设置
persistence true
persistence_location /mosquitto/data/

# 日志设置
log_dest file /mosquitto/log/mosquitto.log
log_type all

# 网络监听
listener 1883
# 如果需要开启 WebSockets，取消下面两行的注释
# listener 9001
# protocol websockets

# 安全认证配置
allow_anonymous false
password_file /mosquitto/config/pwfile
```

#### 3. 编写 Docker Compose 文件
创建 `docker-compose.yml`：

```bash
nano ~/mqtt/docker-compose.yml
```

**粘贴以下内容：**
```yaml
version: '3.8'

services:
  mqtt:
    image: eclipse-mosquitto:latest
    container_name: mqtt-broker
    restart: always
    ports:
      - "1883:1883"
      - "9001:9001"
    volumes:
      - ./config:/mosquitto/config
      - ./data:/mosquitto/data
      - ./log:/mosquitto/log
    environment:
      - TZ=Asia/Shanghai
```

#### 4. 启动并设置账号密码
按照以下顺序操作：

**第一步：启动容器**
```bash
cd ~/mqtt
docker compose up -d
```

**第二步：创建用户 (例如用户名叫 `admin`)**
执行此命令后，系统会提示你输入并确认密码：
```bash
docker exec -it mqtt-broker mosquitto_passwd -b /mosquitto/config/pwfile admin 你的密码
```
*(注意：`-b` 表示在命令行直接输入密码，如果你想交互式输入，请去掉 `-b` 和末尾的密码。)*

**第三步：重启服务使密码生效**
```bash
docker compose restart
```

#### 5. 验证集成结果
现在尝试连接时，必须携带用户名和密码，否则会被拒绝。

* **带密码订阅：**
    ```bash
    docker exec -it mqtt-broker mosquitto_sub -t "test/topic" -u "admin" -P "你的密码"
    ```
    PS：这里运行之后会进入阻塞阶段，放在这里
* **带密码发布：**
    ```bash
    docker exec -it mqtt-broker mosquitto_pub -t "test/topic" -m "Hello with Auth" -u "admin" -P "你的密码"
    ```
    PS：新建一个session，在这里执行之后，就会在上面那个监听session中，看到：Hello with Auth



#### 6. 常用运维操作清单

| 需求 | 命令 |
| :--- | :--- |
| **查看容器日志** | `docker compose logs -f` |
| **新增/修改其他用户** | `docker exec -it mqtt-broker mosquitto_passwd /mosquitto/config/pwfile 新用户名` |
| **删除用户** | `docker exec -it mqtt-broker mosquitto_passwd -D /mosquitto/config/pwfile 用户名` |
| **完全卸载** | `docker compose down -v` (注意：这会删除挂载的数据卷) |

需要我为您演示如何使用 Python 或 MQTTX 客户端连接这个带密码的服务器吗？

### 安装mangoDB

直接通过docker-compose来安装启动单机版的mangoDB
```yaml
services:
  mongodb:
    image: mongo:latest
    container_name: mongodb
    restart: always
    ports:
      - "27017:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: your_user_name
      MONGO_INITDB_ROOT_PASSWORD: your_password
    volumes:
      - mongodb_data:/data/db

volumes:
  mongodb_data:
```

### 安装influxdb

#### 1. 准备目录结构
```bash
mkdir -p ~/influxdb2 && cd ~/influxdb2
```

#### 2. 创建 `docker-compose.yml`
使用编辑器（如 `nano` 或 `vim`）创建文件：
```bash
nano docker-compose.yml
```

将以下内容粘贴进去：

```yaml
# version: '3.8'

services:
  influxdb:
    image: influxdb:latest
    container_name: influxdb
    restart: always
    ports:
      - "8086:8086"
    volumes:
      # 数据持久化
      - ./data:/var/lib/influxdb2
      # 配置文件挂载
      - ./config:/etc/influxdb2
    environment:
      # 初始化配置（可选，也可以在启动后通过 Web 页面手动配置）
      - DOCKER_INFLUXDB_INIT_MODE=setup
      - DOCKER_INFLUXDB_INIT_USERNAME=admin
      - DOCKER_INFLUXDB_INIT_PASSWORD=password123  # 请修改为强密码，至少8位，弱密码启动不了
      - DOCKER_INFLUXDB_INIT_ORG=my-org
      - DOCKER_INFLUXDB_INIT_BUCKET=my-bucket
      - DOCKER_INFLUXDB_INIT_RETENTION=30d
```

### 3. 启动容器
在 `docker-compose.yml` 所在目录下执行：
```bash
docker-compose up -d
```

#### 4. 访问 Web 界面
InfluxDB 2.x 自带功能强大的管理面板。打开浏览器访问：
`http://<你的服务器IP>:8086`
如下图所示，就有一个这样的管理页面，有这个就不需要第三方工具了：
![infuxdb](4f507556/infuxdb.jpg)

如果你在 `docker-compose` 中配置了 `INIT_MODE`，直接用设置的用户名密码登录即可；如果没有配置，页面会引导你进行首次设置。

#### 5. 检查运行状态
```bash
docker ps | grep influxdb
```

#### 6. 常用管理命令
* **查看日志**：`docker logs -f influxdb`
* **进入容器内部**：`docker exec -it influxdb influx --help`
* **停止并移除**：`docker-compose down`（数据会保留在 `./data` 目录）

### 配置seaweedfs
#### 先说一个插曲
本身这部分内容是我第一次尝试构建，简单接触一下分布式存储相关的内容。属于是接触一个新的未知领域。借助Gemini检索的时候遇到了一些插曲：
- 按理本机环境，我是可以不用配置任何鉴权的东西的，但是为了完整体验，我还是配置了。
- 配置完成之后，就所谓遇到了幻觉，配置的鉴权配置文件死活不生效
- 交叉使用DeepSeek专家模式，同样的报错日志，再同样的问一遍，结果得到了完全不一样的回答

- 报错日志：```InvalidAccessKeyId: attempted key 'MY_ADMIN' not found. Available keys: 0, Auth enabled: true```
- Gemini的回答，一直在兜圈子，一直在给我说检查这个配置文件的问题```/data/filer/security.toml```
- Deepseek的回答（截取）：
> 这个错误信息明确指出了以下两点：
> *   **`Auth enabled: true`**：SeaweedFS 的 S3 网关服务（`weed s3`）已经启用了认证，任何请求都必须提供有效的密钥。
> *   **`Available keys: 0`**：服务端**当前没有加载任何有效的访问密钥**，因此它无法识别您请求中提供的密钥 `MY_ADMIN`。
> 
> 所以，根本原因并非密钥错误，而是服务端缺少对应的密钥配置。
> 
> 
> 建议按顺序排查，通常检查配置即可解决。
> 
> 您是否使用了 `-config` 参数来指定配置文件？配置文件是加载密钥的第一步。
> ```bash
> # 您需要类似这样的命令，通过 -config 参数加载配置文件
> weed s3 -filer=localhost:8888 -config=/path/to/your/s3.json
> ```
> 
> SeaweedFS 通过一个 JSON 文件来管理 S3 认证。请确认该文件格式是否正确，并包含了您想用的密钥。
> 
> *   **基本配置格式**：一个正确的配置文件应包含 `identities` 数组，其中定义了用户名、密钥和权限。
>     ```json
>     {
>       "identities": [
>         {
>           "name": "admin_user",
>           "credentials": [
>             {
>               "accessKey": "MY_ADMIN",  // 这是您的访问密钥 ID
>               "secretKey": "your_secret_key_here" // 请替换为您的密钥
>             }
>           ],
>           "actions": ["Admin", "Read", "Write", "List", "Tagging"] // 赋予相应权限
>         }
>       ]
>     }
>     ```
> *   **常见格式错误**：请仔细检查文件，确保 JSON 格式严格正确，特别是不要遗漏或添加多余的逗号、引号，并且路径准确无误。

***此时我发现一个明显的不对劲，Gemini一直跟我说toml文件的问题，而DeepSeek和百度（简单的检索过）都给的明确的配置文件是json格式，于是我问了DeepSeek下面一句：***

> 我配置了toml文件，不行吗？

DeepSeek给了我下面答复：

> 你配置 TOML 文件不行的原因找到了：SeaweedFS 有两种不同的配置系统，它们承担了不同的职责。S3 认证相关的配置（也就是能解决你 `InvalidAccessKeyId` 报错的部分）**必须使用 JSON 格式**，TOML > 文件在这里派不上用场。
> 
> 
> SeaweedFS 使用 TOML 格式来管理其核心组件（如 Filer, Master, Volume）的配置，例如数据库连接、安全证书等。你配置的 TOML 文件很可能就是用于这些方面的，例如 `security.toml`。
> 
> 但问题出在 **S3 网关的认证信息**。这部分的配置是为了定义哪些用户（Access Key/Secret Key）可以访问你的对象存储，和核心组件的 TOML > 配置是完全分开的两套系统，互不通用。这就好比，你用房门钥匙（TOML配置）去开车的后备箱（S3认证），当然打不开。
> 
> 具体区别可以参考下表：
> 
> | 配置类型 | 配置文件格式 | 主要用途 | 能否配置 S3 认证 (Access Key) |
> | :--- | :--- | :--- | :--- |
> | **S3 基础认证** | **JSON** | 定义 S3 用户凭证和基本权限 | **能** |
> | **S3 高级 IAM** | **JSON** | 配置 OIDC、STS 等高级认证 | **能** |
> | **系统核心配置** | **TOML** | 配置各组件（Master、Filer等） | **不能** |
> 
> 
> 要解决 `InvalidAccessKeyId` 错误，你需要为 S3 网关创建一个 **JSON 格式**的配置文件。
> 
> 新建一个文件（比如 `s3.json`），内容如下，把你需要的 `accessKey` 和 `secretKey` 填进去：
> 
> ```json
> {
>   "identities": [
>     {
>       "name": "your_admin_user",
>       "credentials": [
>         {
>           "accessKey": "MY_ADMIN",
>           "secretKey": "your_strong_secret_key_here"
>         }
>       ],
>       "actions": ["Admin", "Read", "Write", "List"]
>     }
>   ]
> }
> ```
> 
> 
> 简单来说，解决这个问题的关键就是：**S3 认证必须用独立的 JSON 配置文件**，你之前配置的 TOML 文件管不到这块。按照上面的方法创建一个正确的 JSON 文件并用 `-config` 参数启动 S3 网关，问题就能解决了。

***吐血！！！回到本质问题，当大模型出现幻觉的时候，你怎么有效的来辨别这个问题的错误？是一个值得思考的问题，有时间单独开一篇写点感悟***

#### 一个最简配置文件
主机映射目录就不单独标注列出了。

```yaml
version: '3.3'

services:
  master:
    image: chrislusf/seaweedfs
    container_name: sw_master
    restart: always
    ports:
      - "19333:9333"
      - "19334:9334"
    volumes:
      - ./data/master:/data
    command: "master -ip=master -port=9333 -mdir=/data"

  volume:
    image: chrislusf/seaweedfs
    container_name: sw_volume
    restart: always
    ports:
      - "18080:8080"
      - "18081:18081"
    volumes:
      - ./data/volume:/data
    depends_on:
      - master
    command: "volume -mserver=master:9333 -port=8080 -dir=/data -max=100"

  filer:
    image: chrislusf/seaweedfs
    container_name: sw_filer
    restart: always
    ports:
      - "18888:8888"
      - "18333:8333"
    volumes:
      - ./data/filer_data:/data
      - ./data/filer:/etc/seaweedfs
    depends_on:
      - master
      - volume
    # 【修正处】：将 -defaultLevelDbDirectory 修改为 -defaultStoreDir
    command: "filer -master=master:9333 -s3 -s3.port=8333 -s3.config=/etc/seaweedfs/iam.json -defaultStoreDir=/data"
```

其中上方，```iam.json``` 鉴权配置文件如下：
实际场景，你完全可以针对你不同的业务应用，单独分配不同权限、不同bucket的secretKey，以做到权限最小使用原则。
*如果不配置这个鉴权配置文件的话，上面的filer的启动命令行中去掉s3.config配置文件即可*
```json
{
  "identities": [
    {
      "name": "admin",
      "credentials": [
        {
          "accessKey": "MY_ADMIN",
          "secretKey": "your_strong_secret_key"
        }
      ],
      "actions": [
        "Read",
        "Write",
        "List",
        "Tagging",
        "Admin"
      ]
    },
    {
      "name": "local_dev",
      "credentials": [
        {
          "accessKey": "LOCAL_DEV",
          "secretKey": "your_strong_secret_key"
        }
      ],
      "actions": [
        "Read",
        "Write",
        "List",
        "Admin"
      ]
    }
  ]
}
```

#### 一段java调用代码
先引入 AWS S3 SDK v2。建议使用 BOM（Bill of Materials）来管理版本。
```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>software.amazon.awssdk</groupId>
            <artifactId>bom</artifactId>
            <version>2.25.10</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <dependency>
        <groupId>software.amazon.awssdk</groupId>
        <artifactId>s3</artifactId>
    </dependency>
</dependencies>
```

java代码：
```java
import software.amazon.awssdk.auth.credentials.AwsBasicCredentials;
import software.amazon.awssdk.auth.credentials.StaticCredentialsProvider;
import software.amazon.awssdk.core.sync.RequestBody;
import software.amazon.awssdk.regions.Region;
import software.amazon.awssdk.services.s3.S3Client;
import software.amazon.awssdk.services.s3.model.*;

import java.net.URI;
import java.nio.file.Paths;

public class S3Service {

    private final S3Client s3Client;
    private final String bucketName = "my-bucket";

    public S3Service() {
        // 1. 配置凭证（SeaweedFS默认可能为空，但在SDK中需填充占位符）
        AwsBasicCredentials credentials = AwsBasicCredentials.create("anyAccessKey", "anySecretKey");

        // 2. 初始化客户端
        this.s3Client = S3Client.builder()
                .endpointOverride(URI.create("http://127.0.0.1:18333")) // 替换为你的服务器IP和S3端口
                .credentialsProvider(StaticCredentialsProvider.create(credentials))
                // .credentialsProvider(AnonymousCredentialsProvider.create()) // 这里如果默认不配置上面的iam.json鉴权配置文件的话，这里直接使用匿名方式访问即可。
                .region(Region.US_EAST_1) // S3协议要求必须有Region，SeaweedFS忽略此值
                .serviceConfiguration(sc -> sc.pathStyleAccessEnabled(true)) // 关键：开启路径样式访问
                .build();
        
        // 初始化创建 Bucket
        ensureBucketExists();
    }

    /**
     * 检查并创建存储桶
     */
    private void ensureBucketExists() {
        try {
            s3Client.headBucket(HeadBucketRequest.builder().bucket(bucketName).build());
        } catch (NoSuchBucketException e) {
            s3Client.createBucket(CreateBucketRequest.builder().bucket(bucketName).build());
        }
    }

    /**
     * 上传文件
     */
    public void uploadFile(String objectKey, String filePath) {
        PutObjectRequest putOb = PutObjectRequest.builder()
                .bucket(bucketName)
                .key(objectKey)
                .build();

        s3Client.putObject(putOb, RequestBody.fromFile(Paths.get(filePath)));
        System.out.println("上传成功: " + objectKey);
    }

    /**
     * 下载文件
     */
    public void downloadFile(String objectKey, String downloadPath) {
        GetObjectRequest getObjectRequest = GetObjectRequest.builder()
                .bucket(bucketName)
                .key(objectKey)
                .build();

        s3Client.getObject(getObjectRequest, Paths.get(downloadPath));
        System.out.println("下载完成: " + downloadPath);
    }

    /**
     * 删除文件
     */
    public void deleteFile(String objectKey) {
        DeleteObjectRequest deleteObjectRequest = DeleteObjectRequest.builder()
                .bucket(bucketName)
                .key(objectKey)
                .build();

        s3Client.deleteObject(deleteObjectRequest);
        System.out.println("删除成功: " + objectKey);
    }

    public static void main(String[] args) {
        S3Service s3 = new S3Service();
        // 示例：将本地的 test.jpg 上传到 OSS 命名为 image/001.jpg
        s3.uploadFile("image/001.jpg", "C:/temp/test.jpg");
    }
}
```

---

## Docker镜像加速地址(持续更新)
```json
{
  "registry-mirrors": [
    "https://docker.m.daocloud.io/",
    "https://hub-mirror.c.163.com",
    "https://dockerproxy.com/",
    "https://mirror.baidubce.com/",
    "https://docker.nju.edu.cn/",
    "https://docker.mirrors.sjtug.sjtu.edu.cn/",
    "https://mirror.ccs.tencentyun.com",
    "https://docker-0.unsee.tech",
    "https://register.liberx.info/",
    "https://docker.registry.cyou/",
    "https://docker-cf.registry.cyou/",
    "https://dockercf.jsdelivr.fyi/",
    "https://docker.jsdelivr.fyi/",
    "https://dockertest.jsdelivr.fyi/",
    "https://mirror.iscas.ac.cn/",
    "https://docker.rainbond.cc/",
    "https://mirror.aliyuncs.com",
    "https://docker.mirrors.ustc.edu.cn/"
  ]
}
```