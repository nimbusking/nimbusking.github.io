---
title: Ubuntu相关系列
mathjax: false
abbrlink: '145818e1'
date: 2019-05-16 02:20:17
tags:
    - Ubuntu
    - Linux
updated: 2025-12-30 02:20:17
categories: 操作系统
---

## 背景
又是一个熬夜的夜晚，凌晨2点半，新晋小备机工作站基本搞定了。 Yeah!

## Ubuntu安装与初始化配置
### 安装
### 机械硬盘挂载
### 切换数据源为阿里云

```shell
#备份source.list
cp -f /etc/apt/sources.list /etc/apt/sources.list.bak
```

通过vi/vim修改sources.list
将原有的都注释了，把下面的数据源新增在文件末尾即可：
```shell
deb http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
```

保存之后，记得：**sudo apt-get update**
更新数据源即可

### 首次登录设置root密码
- 打开终端，输入**sudo passwd root**，
在[sudo] password for wheat:后输入当前用户的密码

- 当前用户密码验证通过后
输入需要设置的root超级管理员账户密码Enter new UNIX password:
验证输入的密码Retype new UNIX password:

- 出现**passwd: password updated successfully**字样，表示超级管理员root账户密码设置成功
- 验证：输入su，后输入超级管理员账户的密码，验证通过则成功切换到root账户root

### 安装jdk并一键切换版本
首次运行下面命令，会显示下面的内容
```shell
kemi@datacenter:~$ java -version
Command 'java' not found, but can be installed with:
sudo apt install openjdk-17-jre-headless  # version 17.0.17+10-1~24.04, or
sudo apt install openjdk-21-jre-headless  # version 21.0.9+10-1~24.04
sudo apt install default-jre              # version 2:1.17-75
sudo apt install openjdk-11-jre-headless  # version 11.0.29+7-1ubuntu1~24.04
sudo apt install openjdk-25-jre-headless  # version 25.0.1+8-1~24.04
sudo apt install openjdk-8-jre-headless   # version 8u472-ga-1~24.04
sudo apt install openjdk-19-jre-headless  # version 19.0.2+7-4
sudo apt install openjdk-20-jre-headless  # version 20.0.2+9-1
sudo apt install openjdk-22-jre-headless  # version 22~22ea-1
```

提示你找不到java命令，需要安装。这里就是ubuntu仓库中各个版本的代号，按需安装即可。例如我默认安装的是jdk8，即：
```shell
sudo apt install openjdk-8-jre-headless
# 再次执行
kemi@datacenter:~$ java -version
openjdk version "1.8.0_472"
OpenJDK Runtime Environment (build 1.8.0_472-8u472-ga-1~24.04-b08)
OpenJDK 64-Bit Server VM (build 25.472-b08, mixed mode)
```

#### 安装多个版本jdk
通过如下命令，可以查看当前安装的jdk版本
```shell
kemi@middileware:~$ sudo dpkg -l | grep 'jdk\|jre'
ii  openjdk-11-jre-headless:amd64        11.0.29+7-1ubuntu1~24.04                amd64        OpenJDK Java runtime, using Hotspot JIT (headless)
ii  openjdk-8-jre-headless:amd64         8u472-ga-1~24.04                        amd64        OpenJDK Java runtime, using Hotspot JIT (headless)
```

通过下述命令，可以切换不同版本的jdk
```shell
kemi@middileware:~$ sudo update-alternatives --config java
There are 2 choices for the alternative java (providing /usr/bin/java).

  Selection    Path                                            Priority   Status
------------------------------------------------------------
  0            /usr/lib/jvm/java-11-openjdk-amd64/bin/java      1111      auto mode
  1            /usr/lib/jvm/java-11-openjdk-amd64/bin/java      1111      manual mode
* 2            /usr/lib/jvm/java-8-openjdk-amd64/jre/bin/java   1081      manual mode

Press <enter> to keep the current choice[*], or type selection number: 
```

**如上所示：我选择了2，则前面会有一个星号，代码当前默认的版本，再次通过java -version可以确认**

#### 配置JAVA_HOME
直接
```shell
sudo vim /etc/environment
# 在下面添加一行即可，具体路径就是上面选择默认版本是第二列中间的
JAVA_HOME="/usr/lib/jvm/java-8-openjdk-amd64/jre/bin/java"
```

配置好之后，记得刷新
```shell
source /etc/environment
echo $JAVA_HOME
```

#### 卸载jdk
- 找到JDK包：`sudo dpkg -l | grep 'jdk\|jre'`
- 使用以下命令卸载这些包，将包名替换为你找到的实际包名：
    ```shell
    sudo apt purge default-jdk default-jdk-headless default-jre default-jre-headless openjdk-21-jdk openjdk-21-jdk-headless openjdk-21-jre openjdk-21-jre-headless
    ```
- 卸载完成后，使用以下命令清除剩余的依赖项：
    ```shell
    sudo apt autoremove --purge
    ```
- 从/etc/environment文件中删除包含JAVA_HOME变量的行，并保存文件。

## 系列话题
### 在ubuntu下编译openjdk
编译方法来自于《深入理解JAVA虚拟机 第三版》，详细请参考该书中第一章，1.6小节的相关说明
#### 准备源码包
根据书中的说明，推荐直接通过网页下载，具体方法如下：
去链接**https://hg.openjdk.java.net/jdk/jdk12/** 访问，点击左侧的**Browse**后，在下方有个zip压缩包，直接点击下载即可。如下图所示：
![下载openjdk12源码压缩包](145818e1/2021-05-08_165324.png)
#### 系统需求

## 相关命令
## 参考